# config

orbs:
  gradle: circleci/gradle@2.2.0
  aws-cli: circleci/aws-cli@3.0.0
  aws-eks: circleci/aws-eks@2.1.2
  aws-ecr: circleci/aws-ecr@8.0.0
  kubernetes: circleci/kubernetes@1.3.0
  jq: circleci/jq@2.2.0
  slack: circleci/slack@4.9.3

# If you want to make a branch deploy to sandbox, just put the branch in here
sandbox_branch_name: &SANDBOX_BRANCH_NAME feat/serv-XXX/not-a-branch

references:
  postgres_environment: &postgres_environment
    environment:
      POSTGRES_USER: "root"
      POSTGRES_DB: "brace"
      POSTGRES_PASSWORD: "u2VMsVTK3DPAa7h+>D6VB#7q8.j7FHHhBd]6dZcY]qnokKVRM{F+BUcNj.B&3oP6"
      POSTGRES_HOSTNAME: "postgres"
  elixir_image: &elixir_image
    image: cimg/elixir:1.13.4-erlang-24.3
    <<: *postgres_environment
  #TODO This may not be necessary because we use the wrapper, we may just need a JVM
  gradle_image: &gradle_image
    image: gradle:7.4-jdk17
  postgres_image: &postgres_image
    image: cimg/postgres:13.4
    name: postgres
    command: ["postgres", "-c", "wal_level=logical"]
    <<: *postgres_environment
  setup_gradle_ci_init: &setup_gradle_ci_init
    run: mkdir -p ~/.gradle && cp gradle/init-with-moved-buildDir.gradle ~/.gradle/init.gradle
  generate_gradle_cache_seed: &generate_gradle_cache_seed
    run: find . -name 'build.gradle.kts' | sort | xargs cat | shasum | awk '{print $1}' > /tmp/gradle_cache_seed
  restore_gradle_cache: &restore_gradle_cache
    restore_cache:
      key: gradle-v1-{{ checksum "/tmp/gradle_cache_seed" }}-{{ checksum ".circleci/config.yml" }}
  save_gradle_cache: &save_gradle_cache
    save_cache:
      paths:
        - "~/.gradle/caches"
        - "~/.gradle/wrapper"
      key: gradle-v1-{{ checksum "/tmp/gradle_cache_seed" }}-{{ checksum ".circleci/config.yml" }}
  workspace_root: &workspace_root /tmp/workspace
  attach_workspace: &attach_workspace
    attach_workspace:
      at: *workspace_root
  shallow_checkout: &SHALLOW_CHECKOUT
    run:
      name: Shallow repository checkout
      command: |
        mkdir -p $HOME/.ssh
        ssh-keyscan -H github.com >> ~/.ssh/known_hosts
        cd ~
        rm -r ~/project
        BRANCH=$CIRCLE_BRANCH
        if [ -z "$BRANCH" ];
        then
          BRANCH=$CIRCLE_TAG
        fi
        git clone --single-branch --depth 1 "$CIRCLE_REPOSITORY_URL" --branch "$BRANCH" ~/project
  slack_notify_failure: &SLACK_NOTIFY_FAIL
    slack/notify:
      branch_pattern: main
      event: fail
      template: basic_fail_1

executors:
  gradle-executor:
    docker:
      - *gradle_image
      - *postgres_image
      - image: cimg/redis:6.2
        name: redis
  psql-and-gradle-executor:
    docker:
      - *gradle_image
      - *postgres_image
  node-executor:
    docker:
      - image: "node:16"

commands:
  backend-test:
    parameters:
      project:
        type: string
      buildSubdir:
        type: string
      test_name:
        type: string
    steps:
      - *SHALLOW_CHECKOUT
      - *attach_workspace
      - *setup_gradle_ci_init
      - *generate_gradle_cache_seed
      - *restore_gradle_cache
      #See https://circleci.com/docs/2.0/databases/#using-dockerize-to-wait-for-dependencies
      - run:
          name: install dockerize
          command: wget https://github.com/jwilder/dockerize/releases/download/$DOCKERIZE_VERSION/dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz && tar -C /usr/local/bin -xzvf dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz && rm dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz
          environment:
            DOCKERIZE_VERSION: v0.6.1
      - run:
          name: Wait for postgres to start up
          command: dockerize -wait tcp://postgres:5432 -timeout 2m
      - run:
          environment:
            ENVIRONMENT: TEST
            IS_CI_CD: "true"
          command: gradle :backend:<< parameters.project >>:<< parameters.test_name >> :backend:<< parameters.project >>:writeSuccessfulOutput --build-cache
      - run:
          name: Ensure that test results were generated
          environment:
            RESULTS_PATH: /tmp/gradle-build/brace/<< parameters.buildSubdir >>/ci-checkpoint/success.txt
          command: stat $RESULTS_PATH
      - *save_gradle_cache
      - gradle/collect_test_results:
          test_results_path: /tmp/gradle-build/brace/<< parameters.buildSubdir >>/test-results
          reports_path: /tmp/gradle-build/brace/<< parameters.buildSubdir >>/reports/tests/<< parameters.test_name >>/
  build-and-push-backend-docker-image:
    parameters:
      project:
        type: string
      override_repo:
        description: This is to allow overriding what the ECR's repo name is because for borrower its not the same as the java project dir name
        type: string
        default: ""
      tag:
        type: string
        default: latest
    steps:
      - *attach_workspace
      - aws-cli/setup
      - setup_remote_docker:
          docker_layer_caching: true
      - run:
          name: Restore shadow tar from workspace
          environment:
            DEST: build/distributions
            ARCHIVE: << parameters.project >>-shadow.tar
          command: |
            mkdir -p ${DEST}
            cp /tmp/workspace/shadow/${ARCHIVE} ${DEST}/
      - run:
          name: Copy in shared binaries
          command: cp -v ../shared-binaries/* .
      - run:
          name: Build docker image
          environment:
            PW_STD_IN: 838607676327.dkr.ecr.us-east-2.amazonaws.com
            REPOSITORY_URL: 838607676327.dkr.ecr.us-east-2.amazonaws.com/brace
            TAG_SUFFIX: << parameters.tag >>
          command: |
            HASH="git-${CIRCLE_SHA1}"
            GIT_DESCRIBE=$(cat /tmp/workspace/git/describe)
            TIMESTAMP_MS=$(date +%s000)
            if [ ! -z "<< parameters.override_repo >>" ];
            then
              REPOSITORY_URL="${REPOSITORY_URL}/<< parameters.override_repo >>"
            else
              REPOSITORY_URL="${REPOSITORY_URL}/<< parameters.project >>"
            fi
            MAIN_TAG=${REPOSITORY_URL}:${TAG_SUFFIX}
            HASH_TAG=${REPOSITORY_URL}:${HASH}
            docker build --tag ${MAIN_TAG} --build-arg build_version=${GIT_DESCRIBE} --build-arg build_timestamp=${TIMESTAMP_MS} .
            docker tag ${MAIN_TAG} ${HASH_TAG}
            aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin ${PW_STD_IN}
            docker push $MAIN_TAG
            docker push $HASH_TAG
  setup-kubectl:
    parameters:
      environment_id:
        description: "The AWS subaccount ID of the environment to target."
        default: 429158029891
        type: integer
      environment_name:
        description: "The environment name, like 'dev'."
        default: "dev"
        type: string
    steps:
      - aws-cli/setup
      - run:
          name: Fixup profile
          environment:
            ROLE_ARN: "arn:aws:iam::<< parameters.environment_id >>:role/OrganizationAccountAccessRole"
          command: |
            echo "[sub-profile]" >> ~/.aws/credentials
            echo "source_profile = default" >> ~/.aws/credentials
            echo "role_arn = ${ROLE_ARN}" >> ~/.aws/credentials
        # Install kubectl explicitly and tag version 1.23.0 due to v1alpha1 ExecCredential api removal
      # https://github.com/kubernetes/sig-release/blob/master/releases/release-1.24/release-notes/release-notes-draft.json#L5477-L5478
      - kubernetes/install-kubectl:
          kubectl-version: v1.23.0
      - aws-eks/update-kubeconfig-with-authenticator:
          install-aws-cli: false
          install-kubectl: false
          cluster-name: << parameters.environment_name >>-brace
          aws-profile: sub-profile
          aws-region: us-east-2
          cluster-authentication-role-arn: "arn:aws:iam::<< parameters.environment_id >>:role/OrganizationAccountAccessRole"
          verbose: true
  autodeploy-service:
    parameters:
      service_name:
        type: string
      environment_name:
        type: string
    steps:
      - run:
          name: Restart service << parameters.service_name >>
          environment:
            SERVICE_NAME: << parameters.service_name >>
            CLUSTER_NAME: << parameters.environment_name >>
          command: kubectl rollout restart -n "${SERVICE_NAME}" deployment/"${SERVICE_NAME}"-${CLUSTER_NAME}
      - run:
          environment:
            EMOJI: ":rocket:"
            SERVICE_NAME: << parameters.service_name >>
            CLUSTER_NAME: << parameters.environment_name >>
          name: Notify slack
          command: ./utilities/notify-slack.sh "${EMOJI} Deploying \`${CIRCLE_SHA1}\` (\`${CIRCLE_BRANCH}\`) to ${SERVICE_NAME} in cluster ${CLUSTER_NAME}"
  rerun-job:
    parameters:
      job_name:
        type: string
      emoji:
        type: string
      environment_name:
        type: string
    steps:
      - run:
          name: Rerun K8s job << parameters.job_name >>
          command: |
            kubectl get job -n jobs "<< parameters.job_name >>" -o json | \
              jq 'del(.spec.template.metadata.labels)' | \
              jq 'del(.spec.selector)' | \
              jq '.spec.parallelism=1' | \
              kubectl replace --force -f -
      - run:
          environment:
            EMOJI: << parameters.emoji >>
            SERVICE_NAME: << parameters.job_name >>
            CLUSTER_NAME: << parameters.environment_name >>
          name: Notify slack
          command: ./utilities/notify-slack.sh "${EMOJI} Running job \`${SERVICE_NAME}\` \`${CIRCLE_SHA1}\` (\`${CIRCLE_BRANCH}\`) in cluster ${CLUSTER_NAME}"
  wait-for-service:
    parameters:
      service_name:
        type: string
      environment_name:
        type: string
      timeout:
        type: integer
        default: 5
        description: "The amount of time to wait for the rollout to finish."
    steps:
      - run:
          name: Wait for << parameters.service_name >> to roll out.
          environment:
            SERVICE_NAME: << parameters.service_name >>
            ENV_NAME: << parameters.environment_name >>
            TIMEOUT: << parameters.timeout >>
          command: kubectl rollout status deployment/${SERVICE_NAME}-${ENV_NAME} -n ${SERVICE_NAME} --timeout ${TIMEOUT}m
  kill-job:
    parameters:
      job_name:
        type: string
    steps:
      - run:
          name: Kill job << parameters.job_name >> if its running
          environment:
            NAME: << parameters.job_name >>
          command: |
            ACTIVE=$(kubectl get jobs -n jobs ${NAME} -o jsonpath='{.status}' | jq '.active')
            [[ "${ACTIVE}" == "null" ]] && { echo "${NAME} is not active: exiting." ; exit 0; }
            kubectl patch job -n jobs ${NAME} -p '{"spec":{"parallelism":0}}'
            ./utilities/notify-slack.sh ":burning-moneybag: Killing unfinished \`${NAME}\` job."
  wait-for-job:
    parameters:
      job_name:
        type: string
      timeout:
        type: integer
        default: 10
        description: "The amount of time to wait for the job to complete."
      namespace:
        description: "The namespace the jobs are in"
        type: string
        default: "jobs"
    steps:
      - run:
          name: Wait for << parameters.job_name >> to complete.
          environment:
            JOB_NAME: << parameters.job_name >>
            TIMEOUT: << parameters.timeout >>
            NAMESPACE: << parameters.namespace >>
          command: kubectl wait --for=condition=complete --timeout ${TIMEOUT}m ${NAMESPACE}/${JOB_NAME} -n ${NAMESPACE}
jobs:
  run-tests:
    parameters:
      env:
        type: string
        default: "dev"
      env_id:
        type: integer
        default: 429158029891
    resource_class: small
    executor: aws-cli/default
    steps:
      - *SHALLOW_CHECKOUT
      - setup-kubectl:
          environment_id: << parameters.env_id >>
          environment_name: << parameters.env >>
      - jq/install
      - rerun-job:
          job_name: "data-delivery-system-test"
          emoji: ":safety_vest:"
          environment_name: << parameters.env >>
      - rerun-job:
          job_name: "etl-system-test"
          emoji: ":safety_vest:"
          environment_name: << parameters.env >>
      #      Disabled e2e auto restart due to resource utilization
      #      - rerun-job:
      #          job_name: "e2e-system-test"
      #          emoji: ":safety_vest:"
      #          environment_name: << parameters.env >>
      - *SLACK_NOTIFY_FAIL
  auto-deploy:
    parameters:
      env:
        type: string
        default: "dev"
      env_id:
        type: integer
        default: 429158029891
    executor: aws-cli/default
    resource_class: small
    steps:
      - *SHALLOW_CHECKOUT
      - setup-kubectl:
          environment_id: << parameters.env_id >>
          environment_name: << parameters.env >>
      - jq/install
      #     Disabled e2e test due to memory utiliziation
      #      - kill-job:
      #          job_name: "e2e-system-test"
      - kill-job:
          job_name: "data-delivery-system-test"
      - kill-job:
          job_name: "etl-system-test"
      - autodeploy-service:
          service_name: "borrower"
          environment_name: << parameters.env >>
      - autodeploy-service:
          service_name: "servicer"
          environment_name: << parameters.env >>
      - autodeploy-service:
          service_name: "async-worker"
          environment_name: << parameters.env >>
      - autodeploy-service:
          service_name: "dispatch"
          environment_name: << parameters.env >>
      - rerun-job:
          job_name: "frontend-deployer-servicer"
          emoji: ":sonic:"
          environment_name: << parameters.env >>
      - rerun-job:
          job_name: "frontend-deployer-borrower"
          emoji: ":sonic:"
          environment_name: << parameters.env >>
      - wait-for-service:
          service_name: "borrower"
          environment_name: << parameters.env >>
      - wait-for-service:
          service_name: "servicer"
          environment_name: << parameters.env >>
      - wait-for-service:
          service_name: "async-worker"
          environment_name: << parameters.env >>
      - wait-for-service:
          service_name: "dispatch"
          environment_name: << parameters.env >>
      - wait-for-job:
          job_name: "frontend-deployer-servicer"
      - wait-for-job:
          job_name: "frontend-deployer-borrower"
      - *SLACK_NOTIFY_FAIL
  push-fe-artifacts:
    parameters:
      tag:
        type: string
        default: "latest"
    executor: aws-cli/default
    resource_class: small
    steps:
      - *attach_workspace
      - aws-cli/setup
      - run:
          name: Overwrite borrower << parameters.tag >> tar in the S3 bucket
          environment:
            ARCHIVE_BUCKET: "ai.brace.dev.frontend.builds"
            ARCHIVE_NAME: "<< parameters.tag >>.tar.gz"
          command: |
            cd /tmp/workspace/borrower
            tar -zcf ${ARCHIVE_NAME} build
            aws s3 cp ${ARCHIVE_NAME} s3://${ARCHIVE_BUCKET}/borrower/${ARCHIVE_NAME}
      - run:
          name: Overwrite servicer << parameters.tag >> tar in the S3 bucket
          environment:
            ARCHIVE_BUCKET: "ai.brace.dev.frontend.builds"
            ARCHIVE_NAME: "<< parameters.tag >>.tar.gz"
          command: |
            cd /tmp/workspace/servicer
            tar -zcf ${ARCHIVE_NAME} build
            aws s3 cp ${ARCHIVE_NAME} s3://${ARCHIVE_BUCKET}/servicer/${ARCHIVE_NAME}
      - *SLACK_NOTIFY_FAIL
  build-push-servicer-image:
    parameters:
      tag:
        type: string
        default: "latest"
    executor: aws-cli/default
    working_directory: ~/project/backend/servicer-api
    steps:
      - *SHALLOW_CHECKOUT
      - build-and-push-backend-docker-image:
          project: "servicer-api"
          tag: << parameters.tag >>
      - *SLACK_NOTIFY_FAIL
  build-push-borrower-image:
    parameters:
      tag:
        type: string
        default: "latest"
    executor: aws-cli/default
    working_directory: ~/project/backend/borrower-api-vertx
    steps:
      - *SHALLOW_CHECKOUT
      - build-and-push-backend-docker-image:
          project: "borrower-api-vertx"
          override_repo: "borrower-api"
          tag: << parameters.tag >>
      - *SLACK_NOTIFY_FAIL
  build-push-async-worker-image:
    parameters:
      tag:
        type: string
        default: "latest"
    executor: aws-cli/default
    working_directory: ~/project/backend/async-worker
    steps:
      - *SHALLOW_CHECKOUT
      - build-and-push-backend-docker-image:
          project: "async-worker"
          tag: << parameters.tag >>
      - *SLACK_NOTIFY_FAIL
  build-push-dispatch-image:
    parameters:
      tag:
        type: string
        default: "latest"
    executor: aws-cli/default
    working_directory: ~/project/backend/dispatch
    steps:
      - *SHALLOW_CHECKOUT
      - *attach_workspace
      - aws-cli/setup
      - setup_remote_docker:
          docker_layer_caching: true
      - run:
          name: Build docker image
          environment:
            REPOSITORY_URL: 838607676327.dkr.ecr.us-east-2.amazonaws.com/brace/dispatch
            PW_STD_IN: 838607676327.dkr.ecr.us-east-2.amazonaws.com
            TAG_SUFFIX: << parameters.tag >>
          command: |
            HASH="git-${CIRCLE_SHA1}"
            GIT_DESCRIBE=$(cat /tmp/workspace/git/describe)
            TIMESTAMP_MS=$(date +%s000)
            MAIN_TAG=${REPOSITORY_URL}:${TAG_SUFFIX}
            HASH_TAG=${REPOSITORY_URL}:${HASH}
            docker build --target build-prod --tag dist:latest .
            docker build --target dist --tag ${MAIN_TAG} --build-arg build_version=${GIT_DESCRIBE} --build-arg build_timestamp=${TIMESTAMP_MS} .
            docker tag ${MAIN_TAG} ${HASH_TAG}
            aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin ${PW_STD_IN}
            docker push $MAIN_TAG
            docker push $HASH_TAG
      - *SLACK_NOTIFY_FAIL
  backend-build:
    docker:
      - *gradle_image
      #Needed for compile time query checking
      - *postgres_image
    resource_class: large
    steps:
      - *SHALLOW_CHECKOUT
      - *attach_workspace
      - *setup_gradle_ci_init
      - *generate_gradle_cache_seed
      - *restore_gradle_cache
      #See https://circleci.com/docs/2.0/databases/#using-dockerize-to-wait-for-dependencies
      - run:
          name: install dockerize
          command: wget https://github.com/jwilder/dockerize/releases/download/$DOCKERIZE_VERSION/dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz && tar -C /usr/local/bin -xzvf dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz && rm dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz
          environment:
            DOCKERIZE_VERSION: v0.6.1
      - run:
          name: Wait for postgres to start up
          command: dockerize -wait tcp://postgres:5432 -timeout 2m
      - run:
          name: Build all shadowDistTars
          environment:
            IS_CI_CD: "true"
          #Gradle automatically pushes this command upward and calls shadowDistTar in all projects from this
          command: |
            export CI_COMMIT_SHA="${CIRCLE_SHA1}"
            export GIT_DESCRIBE=$(cat /tmp/workspace/git/describe)
            gradle shadowDistTar --build-cache --parallel
      - *save_gradle_cache
      - run: |
          mkdir -p /tmp/workspace/shadow
          cd /tmp/gradle-build/brace
          find . -type f -name '*-shadow.tar' -exec cp '{}' /tmp/workspace/shadow ';'
      - run:
          name: Persist DB utils shadowjar (FIXME)
          command: mv /tmp/gradle-build/brace/database-test-utils/libs/* /tmp/workspace/shadow
      - persist_to_workspace:
          root: *workspace_root
          paths:
            - shadow
      - *SLACK_NOTIFY_FAIL
  build-frontend-dependencies:
    executor: node-executor
    working_directory: ~/project/frontend
    steps:
      - *SHALLOW_CHECKOUT
      - *attach_workspace
      - run:
          name: Generate pnpm cache file to checksum
          command: find . -name 'pnpm-lock.yaml' | sort | xargs cat | shasum | awk '{print $1}' > /tmp/pnpm_lock_seed
      - restore_cache:
          keys:
            - pnpm-6-{{ checksum "/tmp/pnpm_lock_seed" }}
      - restore_cache:
          keys:
            - rush-{{ checksum "~/project/frontend/rush.json" }}
      - run: node common/scripts/install-run-rush.js install
      - run: mkdir build
      - run:
          name: Build Dependencies
          command: RUSH_BUILD_CACHE_CREDENTIAL=$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY node common/scripts/install-run-rush.js build -T @brace/borrower-frontend -T @brace/servicer --verbose
      - save_cache:
          key: rush-{{ checksum "~/project/frontend/rush.json" }}
          paths:
            - common/temp/install-run
            - ~/.rush
      - save_cache:
          key: pnpm-6-{{ checksum "/tmp/pnpm_lock_seed" }}
          paths:
            - common/temp/pnpm-store
      - *SLACK_NOTIFY_FAIL
  build-frontends:
    executor: node-executor
    working_directory: ~/project/frontend
    steps:
      - *SHALLOW_CHECKOUT
      - *attach_workspace
      - run:
          name: Generate pnpm cache file to checksum
          command: find . -name 'pnpm-lock.yaml' | sort | xargs cat | shasum | awk '{print $1}' > /tmp/pnpm_lock_seed
      - restore_cache:
          keys:
            - pnpm-6-{{ checksum "/tmp/pnpm_lock_seed" }}
      - restore_cache:
          keys:
            - rush-{{ checksum "~/project/frontend/rush.json" }}
      - run: node common/scripts/install-run-rush.js install
      - run: mkdir build
      - run:
          name: Build Frontends
          command: |
            export BUILD_TIME_STAMP="`date`"
            export GIT_DESCRIBE=$(cat /tmp/workspace/git/describe)
            RUSH_BUILD_CACHE_CREDENTIAL=$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY node common/scripts/install-run-rush.js build -t @brace/borrower-frontend -t @brace/servicer --verbose
            echo "${GIT_DESCRIBE}" > ./clients/borrower-frontend/build/version
            echo "${GIT_DESCRIBE}" > ./clients/servicer/build/version
            echo "" > ./clients/servicer/build/service-worker.js
      - run: mkdir -p /tmp/workspace/borrower && mkdir -p /tmp/workspace/servicer
      - run: mv ./clients/borrower-frontend/build/ /tmp/workspace/borrower/
      - run: mv ./clients/servicer/build/ /tmp/workspace/servicer/
      - save_cache:
          key: rush-{{ checksum "~/project/frontend/rush.json" }}
          paths:
            - common/temp/install-run
            - ~/.rush
      - save_cache:
          key: pnpm-6-{{ checksum "/tmp/pnpm_lock_seed" }}
          paths:
            - common/temp/pnpm-store
      - persist_to_workspace:
          root: *workspace_root
          paths:
            - borrower
            - servicer
            - frontend-borrower
      - *SLACK_NOTIFY_FAIL
  build-servicer:
    executor: gradle-executor
    steps:
      - backend-build:
          project: "servicer-api"
      - *SLACK_NOTIFY_FAIL
  build-borrower:
    executor: gradle-executor
    steps:
      - backend-build:
          project: "borrower-api-vertx"
      - *SLACK_NOTIFY_FAIL
  build-dispatch:
    parallelism: 1
    docker:
      - *elixir_image
    working_directory: ~/project/backend/dispatch
    environment:
      MIX_ENV: test # @TODO this should be a parameter
    steps:
      - *SHALLOW_CHECKOUT
      - run: mix local.hex --force
      - run: mix local.rebar --force
      - restore_cache:
          keys:
            - v4-dispatch-mix-cache-{{ .Branch }}-{{ checksum "mix.lock" }}
            - v4-dispatch-mix-cache-{{ .Branch }}
            - v4-dispatch-mix-cache
      - restore_cache:
          keys:
            - v4-dispatch-build-cache-{{ .Branch }}-{{ checksum "mix.lock" }}
            - v4-dispatch-build-cache-{{ .Branch }}
            - v4-dispatch-build-cache
      - run:
          name: fetch and compile
          command: mix do deps.get, compile
      - save_cache:
          key: v4-dispatch-mix-cache-{{ .Branch }}-{{ checksum "mix.lock" }}
          paths:
            - "deps"
      - save_cache:
          key: v4-dispatch-build-cache-{{ .Branch }}-{{ checksum "mix.lock" }}
          paths:
            - "_build"
      - *SLACK_NOTIFY_FAIL
  build-async-worker:
    executor: gradle-executor
    steps:
      - backend-build:
          project: "async-worker"
      - *SLACK_NOTIFY_FAIL

  borrower-frontend-test-unit:
    executor: node-executor
    working_directory: ~/project/frontend
    steps:
      - *SHALLOW_CHECKOUT
      - *attach_workspace
      - run:
          name: Generate pnpm cache file to checksum
          command: find . -name 'pnpm-lock.yaml' | sort | xargs cat | shasum | awk '{print $1}' > /tmp/pnpm_lock_seed
      - restore_cache:
          keys:
            - pnpm-6-{{ checksum "/tmp/pnpm_lock_seed" }}
      - restore_cache:
          keys:
            - rush-{{ checksum "~/project/frontend/rush.json" }}
      - run:
          name: Install and Build Dependencies
          command: |
            node common/scripts/install-run-rush.js install
            RUSH_BUILD_CACHE_CREDENTIAL=$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY node common/scripts/install-run-rush.js build -T @brace/borrower-frontend
      - run:
          name: Run Unit Tests
          command: |
            cd clients/borrower-frontend
            npm test
      - store_test_results:
          path: clients/borrower-frontend/reports

      - *SLACK_NOTIFY_FAIL

  servicer-frontend-test-unit:
    executor: node-executor
    working_directory: ~/project/frontend
    steps:
      - *SHALLOW_CHECKOUT
      - *attach_workspace
      - run:
          name: Generate pnpm cache file to checksum
          command: find . -name 'pnpm-lock.yaml' | sort | xargs cat | shasum | awk '{print $1}' > /tmp/pnpm_lock_seed
      - restore_cache:
          keys:
            - pnpm-6-{{ checksum "/tmp/pnpm_lock_seed" }}
      - restore_cache:
          keys:
            - rush-{{ checksum "~/project/frontend/rush.json" }}
      - run:
          name: Install and Build Dependencies
          command: |
            node common/scripts/install-run-rush.js install
            RUSH_BUILD_CACHE_CREDENTIAL=$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY node common/scripts/install-run-rush.js build -T @brace/servicer
      - run:
          name: Run Unit Tests
          command: |
            cd clients/servicer
            npm test
      - store_test_results:
          path: clients/servicer/reports

      - *SLACK_NOTIFY_FAIL

  brace-common-test-unit:
    executor: gradle-executor
    steps:
      - backend-test:
          project: "brace-common"
          buildSubdir: "brace-common"
          test_name: "test"
      - *SLACK_NOTIFY_FAIL
  brace-data-test-all:
    executor: psql-and-gradle-executor
    steps:
      - backend-test:
          project: "brace-data"
          buildSubdir: "brace-data"
          test_name: "testAll"
      - *SLACK_NOTIFY_FAIL
  steep-test:
    executor: psql-and-gradle-executor
    steps:
      - backend-test:
          project: "steep"
          buildSubdir: "steep"
          test_name: "test"
      - *SLACK_NOTIFY_FAIL
  query-gen-test:
    executor: psql-and-gradle-executor
    steps:
      - backend-test:
          project: "query-processor:query-annotation-processor"
          buildSubdir: "query-annotation-processor"
          test_name: "test"
      - *SLACK_NOTIFY_FAIL
  database-test-utils-test-all:
    executor: psql-and-gradle-executor
    steps:
      - backend-test:
          project: "database-test-utils"
          buildSubdir: "database-test-utils"
          test_name: "testAll"
      - *SLACK_NOTIFY_FAIL
  vertx-utils-test:
    executor: gradle-executor
    t:
    executor: gradle-executor
    steps:
      - backend-test:
          project: "vertx-test-utils"
          buildSubdir: "vertx-test-utils"
          test_name: "test"
      - *SLACK_NOTIFY_FAIL
  async-worker-test-unit:
    executor: gradle-executor
    steps:
      - backend-test:
          project: "async-worker"
          buildSubdir: "async-worker"
          test_name: "test"
      - *SLACK_NOTIFY_FAIL
  async-worker-test-integration:
    executor: gradle-executor
    steps:
      - backend-test:
          project: "async-worker"
          buildSubdir: "async-worker"
          test_name: "testIntegration"
      - *SLACK_NOTIFY_FAIL
  servicer-api-test-unit:
    executor: gradle-executor
    steps:
      - backend-test:
          project: "servicer-api"
          buildSubdir: "servicer-api"
          test_name: "test"
      - *SLACK_NOTIFY_FAIL
  servicer-api-test-integration:
    executor: gradle-executor
    steps:
      - backend-test:
          project: "servicer-api"
          buildSubdir: "servicer-api"
          test_name: "testIntegration"
      - *SLACK_NOTIFY_FAIL
  servicer-api-test-qc-waterfall:
    executor: gradle-executor
    steps:
      - backend-test:
          project: "servicer-api"
          buildSubdir: "servicer-api"
          test_name: "testQcWaterfall"
      - *SLACK_NOTIFY_FAIL
  servicer-api-test-cucumber:
    executor: gradle-executor
    steps:
      - backend-test:
          project: "servicer-api"
          buildSubdir: "servicer-api"
          test_name: "testCucumber"
      - *SLACK_NOTIFY_FAIL
  tasks-test-unit:
    executor: gradle-executor
    steps:
      - backend-test:
          project: "tasks"
          buildSubdir: "tasks"
          test_name: "test"
      - *SLACK_NOTIFY_FAIL
  tasks-test-integration:
    executor: gradle-executor
    steps:
      - backend-test:
          project: "tasks"
          buildSubdir: "tasks"
          test_name: "testIntegration"
      - *SLACK_NOTIFY_FAIL
  rules-engine-test-unit:
    executor: gradle-executor
    steps:
      - backend-test:
          project: "rules-engine"
          buildSubdir: "rules-engine"
          test_name: "test"
      - *SLACK_NOTIFY_FAIL
  rules-engine-test-integration:
    executor: gradle-executor
    steps:
      - backend-test:
          project: "rules-engine"
          buildSubdir: "rules-engine"
          test_name: "testIntegration"
      - *SLACK_NOTIFY_FAIL
  borrower-api-test-unit:
    executor: gradle-executor
    steps:
      - backend-test:
          project: "borrower-api-vertx"
          buildSubdir: "borrower-api-vertx"
          test_name: "test"
      - *SLACK_NOTIFY_FAIL
  borrower-api-test-integration:
    executor: gradle-executor
    steps:
      - backend-test:
          project: "borrower-api-vertx"
          buildSubdir: "borrower-api-vertx"
          test_name: "testIntegration"
      - *SLACK_NOTIFY_FAIL
  oas3-annotation-processor-test:
    executor: gradle-executor
    steps:
      - backend-test:
          project: "oas3-annotationprocessor"
          buildSubdir: "oas3-annotationprocessor"
          test_name: "test"
      - *SLACK_NOTIFY_FAIL
  borrower-e2e-test:
    machine:
      docker_layer_caching: true
    steps:
      - checkout:
          path: ~/project
      - run: |
          echo 'export NVM_DIR="/opt/circleci/.nvm"' >> $BASH_ENV
          echo ' [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"' >> $BASH_ENV
      - run: |
          node -v
      - run: |
          nvm install v16
          node -v
          nvm alias default v16
      - run: |
          node -v
      - *attach_workspace
      - run:
          name: Build E2E image
          command: |
            docker build -t e2e-test-ci:latest -f docker/tests/e2e/Dockerfile .
      - run:
          name: Start Borrower and Servicer API
          command: |
            docker-compose -f docker/docker-compose-ci.yml up -d borrower-api servicer-api && \
            ./utilities/wait-for-http.sh localhost:8080 Borrower-API && \
            ./utilities/wait-for-http.sh localhost:8081 Servicer-API
      - run:
          name: Run UI and E2E Test
          command: |
            echo "Warn: Starting Borrower-UI outside of docker compose due to pnpm and docker compatibility!"
            cd /tmp/workspace/frontend-borrower/clients/borrower-frontend && \
            API_HOST=localhost REACT_APP_API_BASE_URL=/api nohup npm start &> borrower-ui.log & \
            ~/project/utilities/wait-for-http.sh https://localhost:3000 Borrower-UI && \
            docker run \
              --network=host \
              -v ~/.gradle/caches:/root/.gradle/caches \
              -v ~/e2e-reports:/app/reports \
              -e BORROWER_API_URL=http://localhost:8080 \
              -e BORROWER_UI_URL=https://localhost:3000 \
              -e SERVICER_API_URL=http://localhost:8081 \
              -e SERVICER_UI_URL=https://localhost:3001 \
              -e AWS_REGION=${AWS_DEFAULT_REGION} \
              -e AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID} \
              -e AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY} \
              -w=/app \
              e2e-test-ci:latest npm run test:headless -- --spec ./test/specs/brace/mock-passing-test.ts
  test-dispatch:
    parallelism: 1
    docker:
      - *elixir_image
      - *postgres_image
    working_directory: ~/project/backend/dispatch
    environment:
      MIX_ENV: test
    steps:
      - *SHALLOW_CHECKOUT
      - run:
          name: install flyway
          working_directory: ~
          command: wget -qO- https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/8.2.3/flyway-commandline-8.2.3-linux-x64.tar.gz | tar xvz && sudo ln -s `pwd`/flyway-8.2.3/flyway ~
      - run:
          name: run migrations
          working_directory: ~/project/backend/database-migrations
          command: ~/flyway -password=${POSTGRES_PASSWORD} -configFiles=./flyway-ci.conf migrate
          environment:
            GIT_BRANCH: << pipeline.git.branch >>
      - run: mix local.hex --force
      - run: mix local.rebar --force
      - restore_cache:
          keys:
            - v4-dispatch-mix-cache-{{ .Branch }}-{{ checksum "mix.lock" }}
            - v4-dispatch-mix-cache-{{ .Branch }}
            - v4-dispatch-mix-cache
      - restore_cache:
          keys:
            - v4-dispatch-build-cache-{{ .Branch }}-{{ checksum "mix.lock" }}
            - v4-dispatch-build-cache-{{ .Branch }}
            - v4-dispatch-build-cache
      - run:
          name: fetch and compile
          command: mix do deps.get, compile
      - run:
          name: Test
          command: mix test
      - *SLACK_NOTIFY_FAIL
  lint-dispatch:
    parallelism: 1
    docker:
      - *elixir_image
    working_directory: ~/project/backend/dispatch
    environment:
      MIX_ENV: test
    steps:
      - *SHALLOW_CHECKOUT
      - run: mix local.hex --force
      - run: mix local.rebar --force
      - restore_cache:
          keys:
            - v4-dispatch-mix-cache-{{ .Branch }}-{{ checksum "mix.lock" }}
            - v4-dispatch-mix-cache-{{ .Branch }}
            - v4-dispatch-mix-cache
      - restore_cache:
          keys:
            - v4-dispatch-build-cache-{{ .Branch }}-{{ checksum "mix.lock" }}
            - v4-dispatch-build-cache-{{ .Branch }}
            - v4-dispatch-build-cache
      - run:
          name: fetch
          command: mix do clean, deps.get
      - run:
          name: Compile with warnings as errors
          command: mix compile --warnings-as-errors
      - run:
          name: Formatting Check
          command: mix format --check-formatted
      - run:
          name: Credo Linter
          command: mix credo --strict
      - *SLACK_NOTIFY_FAIL
  dialyzer-dispatch:
    parallelism: 1
    docker:
      - *elixir_image
    working_directory: ~/project/backend/dispatch
    environment:
      MIX_ENV: dev # let's not check the types in tests
    steps:
      - *SHALLOW_CHECKOUT
      - run: mix local.hex --force
      - run: mix local.rebar --force
      - restore_cache:
          keys:
            - v4-dispatch-mix-cache-{{ .Branch }}-{{ checksum "mix.lock" }}
            - v4-dispatch-mix-cache-{{ .Branch }}
            - v4-dispatch-mix-cache
      - restore_cache:
          keys:
            - v4-dispatch-dialyzer-cache-{{ .Branch }}-{{ checksum "mix.lock" }}
            - v4-dispatch-dialyzer-cache-{{ .Branch }}
            - v4-dispatch-dialyzer-cache
      - run: mix do deps.get, compile
      - run:
          name: Dialyzer Caching
          command: mix dialyzer --plt
      - run:
          name: Dialyzer Type Checker
          command: mix dialyzer
      - save_cache:
          key: v4-dispatch-dialyzer-cache-{{ .Branch }}-{{ checksum "mix.lock" }}
          paths:
            - "_build"
      - *SLACK_NOTIFY_FAIL
  build-push-etl-system-test:
    parameters:
      tag:
        type: string
        default: "latest"
    executor: aws-cli/default
    steps:
      - *SHALLOW_CHECKOUT
      - aws-cli/setup
      - setup_remote_docker:
          docker_layer_caching: true
      - run:
          name: Build docker image
          environment:
            MAIN_TAG: 838607676327.dkr.ecr.us-east-2.amazonaws.com/brace/etl-system-test:<< parameters.tag >>
            PW_STD_IN: 838607676327.dkr.ecr.us-east-2.amazonaws.com
          command: |
            DEV_BUILD_VERSION=$(git rev-parse HEAD)
            IMESTAMP_MS=$(date +%s000)
            docker build \
              --tag ${MAIN_TAG} \
              --build-arg build_version=${DEV_BUILD_VERSION} --build-arg build_timestamp=${TIMESTAMP_MS} \
              -f docker/tests/etl/Dockerfile .
            aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin ${PW_STD_IN}
            docker push $MAIN_TAG
      - *SLACK_NOTIFY_FAIL

  build-push-data-delivery-system-test:
    parameters:
      tag:
        type: string
        default: "latest"
    executor: aws-cli/default
    steps:
      - *SHALLOW_CHECKOUT
      - aws-cli/setup
      - setup_remote_docker:
          docker_layer_caching: true
      - run:
          name: Build docker image
          environment:
            MAIN_TAG: 838607676327.dkr.ecr.us-east-2.amazonaws.com/brace/data-delivery-system-test:<< parameters.tag >>
            PW_STD_IN: 838607676327.dkr.ecr.us-east-2.amazonaws.com
          command: |
            DEV_BUILD_VERSION=$(git rev-parse HEAD)
            IMESTAMP_MS=$(date +%s000)
            docker build \
              --tag ${MAIN_TAG} \
              --build-arg build_version=${DEV_BUILD_VERSION} --build-arg build_timestamp=${TIMESTAMP_MS} \
              -f docker/tests/data-delivery/Dockerfile .
            aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin ${PW_STD_IN}
            docker push $MAIN_TAG
      - *SLACK_NOTIFY_FAIL

  build-push-e2e-system-test:
    parameters:
      tag:
        type: string
        default: "latest"
    executor: aws-cli/default
    steps:
      - *SHALLOW_CHECKOUT
      - *attach_workspace
      - aws-cli/setup
      - setup_remote_docker:
          docker_layer_caching: true
      - run:
          name: Grab the jars
          command: |
            mkdir -p jars
            cp --verbose /tmp/workspace/shadow/*.jar jars/
      - run:
          name: Build docker image
          environment:
            MAIN_TAG: 838607676327.dkr.ecr.us-east-2.amazonaws.com/brace/e2e-system-test:<< parameters.tag >>
            PW_STD_IN: 838607676327.dkr.ecr.us-east-2.amazonaws.com
          command: |
            DEV_BUILD_VERSION=$(git rev-parse HEAD)
            IMESTAMP_MS=$(date +%s000)
            docker build \
              --tag ${MAIN_TAG} \
              --build-arg build_version=${DEV_BUILD_VERSION} --build-arg build_timestamp=${TIMESTAMP_MS} \
              -f docker/tests/e2e/Dockerfile .
            aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin ${PW_STD_IN}
            docker push $MAIN_TAG
      - *SLACK_NOTIFY_FAIL

  pre-process:
    resource_class: small
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - *attach_workspace
      - run:
          name: Make a workspace checkout for git metadata
          command: mkdir /tmp/workspace/git
      - run:
          name: Enforce branch naming conventions
          command: ./.circleci/check-branch-name.sh
      - run:
          name: Enforce commit message conventions
          command: ./.circleci/check-commit-messages.sh
      - run:
          name: Save the output of git describe --tags
          command: git describe --tags > /tmp/workspace/git/describe
      - run:
          name: Cancel redundant pipelines regardless of user
          command: ./.circleci/cancel-redundant-builds.sh
      - persist_to_workspace:
          root: *workspace_root
          paths:
            - git
      - *SLACK_NOTIFY_FAIL

completion_jobs: &COMPLETION_JOBS
  - brace-common-test-unit
  - brace-data-test-all
  - steep-test
  - query-gen-test
  - vertx-utils-test
  - vertx-test-utils-test
  - servicer-api-test-unit
  - servicer-api-test-integration
  - servicer-api-test-qc-waterfall
  - servicer-api-test-cucumber
  - tasks-test-unit
  - tasks-test-integration
  - rules-engine-test-unit
  - rules-engine-test-integration
  - database-test-utils-test-all
  - borrower-api-test-unit
  - borrower-api-test-integration
  - async-worker-test-unit
  - async-worker-test-integration
  - oas3-annotation-processor-test
  - test-dispatch
  - lint-dispatch
  - build-frontends
  # @TODO: Reenable when this is fixed
  # - borrower-frontend-test-unit
  - servicer-frontend-test-unit

all_context: &ALL_CONTEXT
  context:
    - aws-secrets
    - slack-application

build_job_cfg: &BUILD_JOB_CFG
  <<: *ALL_CONTEXT
  filters:
    tags:
      only: /^v.*/
  requires:
    - pre-process

backend_job_cfg: &BACKEND_TEST_JOB_CFG
  <<: *BUILD_JOB_CFG
  requires:
    - backend-build

dispatch_job_cfg: &DISPATCH_TEST_JOB_CFG
  <<: *BUILD_JOB_CFG
  requires:
    - build-dispatch

only_main: &ONLY_MAIN
  <<: *ALL_CONTEXT
  requires: *COMPLETION_JOBS
  filters:
    branches:
      only:
        - main

e2e_job_cfg: &E2E_JOB_CFG
  <<: *ONLY_MAIN
  requires:
    - backend-build
    - build-frontends
    # - borrower-frontend-test-unit
    - servicer-frontend-test-unit

frontend-build: &FE_BUILD
  <<: *BUILD_JOB_CFG
  requires:
    - build-frontend-dependencies

frontend-test-unit: &FRONTEND_TEST_UNIT
  <<: *ALL_CONTEXT
  requires:
    - build-frontend-dependencies

only_sandbox: &ONLY_SANDBOX
  <<: *ALL_CONTEXT
  tag: sandbox
  requires: *COMPLETION_JOBS
  filters:
    branches:
      only:
        - *SANDBOX_BRANCH_NAME

only_tag: &ONLY_TAG
  <<: *ALL_CONTEXT
  tag: << pipeline.git.tag >>
  requires: *COMPLETION_JOBS
  filters:
    branches:
      ignore: /.*/
    tags:
      only: /^v.*/

workflows:
  build_and_deploy:
    jobs:
      ##### Before all

      - pre-process:
          filters:
            tags:
              only: /^v.*/
          context:
            - slack-application
            - circle-api-key

      ##### Building

      - build-dispatch:
          <<: *BUILD_JOB_CFG
      - dialyzer-dispatch:
          <<: *BUILD_JOB_CFG
      - backend-build:
          <<: *BUILD_JOB_CFG
      - build-frontend-dependencies:
          <<: *BUILD_JOB_CFG

      ##### FE builds are dependent on FE deps

      - build-frontends:
          <<: *FE_BUILD

      ###### Testing

      - brace-common-test-unit:
          <<: *BACKEND_TEST_JOB_CFG
      - steep-test:
          <<: *BACKEND_TEST_JOB_CFG
      - query-gen-test:
          <<: *BACKEND_TEST_JOB_CFG
      - vertx-utils-test:
          <<: *BACKEND_TEST_JOB_CFG
      - vertx-test-utils-test:
          <<: *BACKEND_TEST_JOB_CFG
      - async-worker-test-unit:
          <<: *BACKEND_TEST_JOB_CFG
      - async-worker-test-integration:
          <<: *BACKEND_TEST_JOB_CFG
      - servicer-api-test-unit:
          <<: *BACKEND_TEST_JOB_CFG
      - servicer-api-test-integration:
          <<: *BACKEND_TEST_JOB_CFG
      - servicer-api-test-qc-waterfall:
          <<: *BACKEND_TEST_JOB_CFG
      - servicer-api-test-cucumber:
          <<: *BACKEND_TEST_JOB_CFG
      - tasks-test-unit:
          <<: *BACKEND_TEST_JOB_CFG
      - tasks-test-integration:
          <<: *BACKEND_TEST_JOB_CFG
      - rules-engine-test-unit:
          <<: *BACKEND_TEST_JOB_CFG
      - rules-engine-test-integration:
          <<: *BACKEND_TEST_JOB_CFG
      - database-test-utils-test-all:
          <<: *BACKEND_TEST_JOB_CFG
      - brace-data-test-all:
          <<: *BACKEND_TEST_JOB_CFG
      - borrower-api-test-unit:
          <<: *BACKEND_TEST_JOB_CFG
      - borrower-api-test-integration:
          <<: *BACKEND_TEST_JOB_CFG
      - oas3-annotation-processor-test:
          <<: *BACKEND_TEST_JOB_CFG
      - test-dispatch:
          <<: *DISPATCH_TEST_JOB_CFG
      - lint-dispatch:
          <<: *DISPATCH_TEST_JOB_CFG
      # @TODO: Disabled May 11, 2022 - re-enable after figuring out whats wrong.
      # https://app.circleci.com/pipelines/github/brace-ai/brace-core/6262/workflows/9eefbd79-413a-4114-85c1-8930386c55a7/jobs/109392/steps
      # - borrower-frontend-test-unit:
      #     <<: *BORROWER_FRONTEND_TEST_UNIT
      - servicer-frontend-test-unit:
          <<: *FRONTEND_TEST_UNIT

      ###### Build and push main images

      - build-push-dispatch-image:
          <<: *ONLY_MAIN
          name: build-push-dispatch-image-main
      - build-push-borrower-image:
          <<: *ONLY_MAIN
          name: build-push-borrower-image-main
      - build-push-servicer-image:
          <<: *ONLY_MAIN
          name: build-push-servicer-image-main
      - build-push-async-worker-image:
          <<: *ONLY_MAIN
          name: build-push-async-worker-image-main
      - push-fe-artifacts:
          <<: *ONLY_MAIN
          name: push-fe-artifacts-main
      - build-push-etl-system-test:
          <<: *ONLY_MAIN
          name: build-push-etl-system-test-main
      - build-push-data-delivery-system-test:
          <<: *ONLY_MAIN
          name: build-push-data-delivery-system-test-main
      - build-push-e2e-system-test:
          <<: *ONLY_MAIN
          name: build-push-e2e-system-test-main

      ###### Build and push sandbox images

      - build-push-dispatch-image:
          <<: *ONLY_SANDBOX
          name: build-push-dispatch-image-sandbox
      - build-push-borrower-image:
          <<: *ONLY_SANDBOX
          name: build-push-borrower-image-sandbox
      - build-push-servicer-image:
          <<: *ONLY_SANDBOX
          name: build-push-servicer-image-sandbox
      - build-push-async-worker-image:
          <<: *ONLY_SANDBOX
          name: build-push-async-worker-image-sandbox
      - push-fe-artifacts:
          <<: *ONLY_SANDBOX
          name: push-fe-artifacts-sandbox
      - build-push-etl-system-test:
          <<: *ONLY_SANDBOX
          name: build-push-etl-system-test-sandbox
      - build-push-data-delivery-system-test:
          <<: *ONLY_SANDBOX
          name: build-push-data-delivery-system-test-sandbox
      - build-push-e2e-system-test:
          <<: *ONLY_SANDBOX
          name: build-push-e2e-system-test-sandbox

      ###### Build and push tag images

      - build-push-dispatch-image:
          <<: *ONLY_TAG
          name: build-push-dispatch-image-tag
      - build-push-borrower-image:
          <<: *ONLY_TAG
          name: build-push-borrower-image-tag
      - build-push-servicer-image:
          <<: *ONLY_TAG
          name: build-push-servicer-image-tag
      - build-push-async-worker-image:
          <<: *ONLY_TAG
          name: build-push-async-worker-image-tag
      - push-fe-artifacts:
          <<: *ONLY_TAG
          name: push-fe-artifacts-tag
      - build-push-etl-system-test:
          <<: *ONLY_TAG
          name: build-push-etl-system-test-tag
      - build-push-data-delivery-system-test:
          <<: *ONLY_TAG
          name: build-push-data-delivery-system-test-tag
      - build-push-e2e-system-test:
          <<: *ONLY_TAG
          name: build-push-e2e-system-test-tag
      # @TODO: Disabled Dec 27, 2021 - re-enable after figuring out whats wrong.
      # https://app.circleci.com/pipelines/github/brace-ai/brace-core/1701/workflows/6444286f-4b0f-4147-bde8-25dbb1028e27/jobs/6677
      # - borrower-e2e-test:
      #    <<: *E2E_JOB_CFG

      ###### Auto-deploy built docker images to dev and sandbox

      - auto-deploy:
          <<: *ALL_CONTEXT
          name: auto-deploy-main
          filters:
            branches:
              only:
                - main
          requires:
            - build-push-dispatch-image-main
            - build-push-borrower-image-main
            - build-push-servicer-image-main
            - build-push-async-worker-image-main
            - push-fe-artifacts-main
            - build-push-etl-system-test-main
            - build-push-data-delivery-system-test-main
            - build-push-e2e-system-test-main
            # @TODO: Reenable when this is fixed
            # - borrower-e2e-test
      - auto-deploy:
          <<: *ALL_CONTEXT
          name: auto-deploy-sandbox
          env: "sandbox"
          env_id: 768557322045
          filters:
            branches:
              only:
                - *SANDBOX_BRANCH_NAME
          requires:
            - build-push-dispatch-image-sandbox
            - build-push-borrower-image-sandbox
            - build-push-servicer-image-sandbox
            - build-push-async-worker-image-sandbox
            - push-fe-artifacts-sandbox
            - build-push-etl-system-test-sandbox
            - build-push-data-delivery-system-test-sandbox
            - build-push-e2e-system-test-sandbox

      ###### Initiate System Tests when deployment is successful

      - run-tests:
          <<: *ALL_CONTEXT
          name: run-tests-main
          filters:
            branches:
              only:
                - main
          requires:
            - auto-deploy-main

      - run-tests:
          <<: *ALL_CONTEXT
          name: run-tests-sandbox
          env: "sandbox"
          env_id: 768557322045
          filters:
            branches:
              only:
                - *SANDBOX_BRANCH_NAME
          requires:
            - auto-deploy-sandbox
