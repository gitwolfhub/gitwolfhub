#!/bin/bash

# Ignore iOS error if there is no paid plan
if [ "$CIRCLE_JOB" == "ios" ] && [ "$HAS_PAID_PLAN" != 1 ]; then
  echo "No paid plan, not calling webhook"
  exit
fi

set -euo pipefail

# Script based on https://altinukshini.wordpress.com/2019/01/09/circleci-notifications-in-rocketchat/

url="https://$WEBHOOK_HOSTNAME/api/v1/github-repos/$PROJECT_ID/circleci_webhook/"

payload=$(
cat <<EOM
{
    "status": "$1",
    "job": "$CIRCLE_JOB",
    "build_num": "$CIRCLE_BUILD_NUM",
    "repo": "$CIRCLE_REPOSITORY_URL",
    "branch": "$CIRCLE_BRANCH",
    "build_url": "$CIRCLE_BUILD_URL",
    "compare_url": "$CIRCLE_COMPARE_URL",
    "sha1": "$CIRCLE_SHA1"
}
EOM
)

curl -X POST -H "Content-Type: application/json" -H "Authorization: Api-Key $WEBHOOK_API_KEY" --data "$payload" $url
echo "Webhook call completed"
version: 2
jobs:
  node:
    working_directory: ~/build
    docker:
      - image: circleci/node:10
    steps:
      - checkout

      - restore_cache:
          key: yarn-v1-{{ checksum "yarn.lock" }}-{{ arch }}

      - restore_cache:
          key: node-v1-{{ checksum "package.json" }}-{{ arch }}

      - run: yarn install

      - save_cache:
          key: yarn-v1-{{ checksum "yarn.lock" }}-{{ arch }}
          paths:
            - ~/.cache/yarn

      - save_cache:
          key: node-v1-{{ checksum "package.json" }}-{{ arch }}
          paths:
            - node_modules

      #- run:
      #    name: jest tests
      #    command: |
      #      mkdir -p test-results/jest
      #      yarn run test
      #    environment:
      #      JEST_JUNIT_OUTPUT: test-results/jest/junit.xml

      - persist_to_workspace:
          root: ~/build
          paths:
            - node_modules

      #- store_test_results:
      #    path: test-results

      #- store_artifacts:
      #    path: test-results

      - run:
          name: Webhook Failed
          command: bash .circleci/webhook_callback.sh "failure"
          when: on_fail

  android:
    working_directory: ~/build
    docker:
      - image: circleci/android:api-28-node
    steps:
      - checkout:
          path: ~/build

      - attach_workspace:
          at: ~/build

      - restore_cache:
          key: bundle-v1-{{ checksum "Gemfile.lock" }}-{{ arch }}

      - run:
          command: bundle install
          working_directory: android

      - save_cache:
          key: bundle-v1-{{ checksum "Gemfile.lock" }}-{{ arch }}
          paths:
            - vendor/bundle

      #- run:
      #    name: Run tests
      #    command: |
      #      mkdir -p test-results/fastlane
      #      bundle exec fastlane test
      #      mv fastlane/report.xml test-results/fastlane

      #- store_test_results:
      #    path: test-results

      #- store_artifacts:
      #    path: test-results

      - run:
          name: Build and upload to appetize.io
          command: bundle exec fastlane deploy_appetize
          working_directory: android

      - store_artifacts:
          path: android/app/build/outputs/apk/release/app-release.apk

      - run:
          name: Webhook Success
          command: bash .circleci/webhook_callback.sh "success"
          when: on_success

      - run:
          name: Webhook Failed
          command: bash .circleci/webhook_callback.sh "failure"
          when: on_fail

  ios:
    macos:
      xcode: "11.1.0"
    working_directory: ~/build

    # use a --login shell so our "set Ruby version" command gets picked up for later steps
    shell: /bin/bash --login -o pipefail

    steps:
      - add_ssh_keys:
          fingerprints:
            - "c5:bd:6d:0b:a1:39:97:83:ca:b5:13:59:14:ff:49:40"

      - checkout

      - run:
          name: Check if app has a paid plan
          command: |
            if [ "$HAS_PAID_PLAN" != 1 ]; then
              exit 1
            fi

      - run:
          name: set Ruby version
          command: echo "ruby-2.5" > ~/.ruby-version

      - restore_cache:
          key: yarn-v1-{{ checksum "yarn.lock" }}-{{ arch }}

      - restore_cache:
          key: node-v1-{{ checksum "package.json" }}-{{ arch }}

      # not using a workspace here as Node and Yarn versions
      # differ between our macOS executor image and the Docker containers above
      - run: yarn install

      - save_cache:
          key: yarn-v1-{{ checksum "yarn.lock" }}-{{ arch }}
          paths:
            - ~/.cache/yarn

      - save_cache:
          key: node-v1-{{ checksum "package.json" }}-{{ arch }}
          paths:
            - node_modules

      - restore_cache:
          key: bundle-v1-{{ checksum "ios/Gemfile.lock" }}-{{ arch }}

      - run:
          command: bundle install
          working_directory: ios

      - save_cache:
          key: bundle-v1-{{ checksum "ios/Gemfile.lock" }}-{{ arch }}
          paths:
            - vendor/bundle

      - restore_cache:
          key: 1-pods-{{ checksum "ios/Podfile.lock" }}
          paths:
            - ios/Pods

      - run:
          name: Update CocaPods dependencies
          command: pod install --verbose
          working_directory: ios
          timeout: 1200

      - save_cache:
          key: 1-pods-{{ checksum "ios/Podfile.lock" }}
          paths:
            - ios/Pods

      - run:
          name: Run tests
          command: bundle exec fastlane tests
          working_directory: ios

      - run:
          name: Set up test results
          working_directory: ios
          command: |
            mkdir -p test-results/fastlane test-results/xcode
            mv fastlane/report.xml test-results/fastlane
            mv fastlane/test_output/report.junit test-results/xcode/junit.xml

      #- store_test_results:
      #    path: ios/test-results

      #- store_artifacts:
      #    path: ios/test-results

      - run:
          name: Build and upload to appetize.io
          command: bundle exec fastlane deploy_appetize
          working_directory: ios

      - store_artifacts:
          path: /tmp/fastlane_build/app.zip

      - run:
          name: Webhook Success
          command: bash .circleci/webhook_callback.sh "success"
          when: on_success

      - run:
          name: Webhook Failed
          command: bash .circleci/webhook_callback.sh "failure"
          when: on_fail

workflows:
  version: 2
  node-android-ios:
    jobs:
      - node
      - android:
          requires:
            - node
      - ios:
          requires:
            - node
"""mineblox_18653 URL Configuration

The `urlpatterns` list routes URLs to views. For more information please see:
    https://docs.djangoproject.com/en/2.2/topics/http/urls/
Examples:
Function views
    1. Add an import:  from my_app import views
    2. Add a URL to urlpatterns:  path('', views.home, name='home')
Class-based views
    1. Add an import:  from other_app.views import Home
    2. Add a URL to urlpatterns:  path('', Home.as_view(), name='home')
Including another URLconf
    1. Import the include() function: from django.urls import include, path
    2. Add a URL to urlpatterns:  path('blog/', include('blog.urls'))
"""
from django.contrib import admin
from django.urls import path, include
from allauth.account.views import confirm_email
from rest_framework import permissions
from drf_yasg.views import get_schema_view
from drf_yasg import openapi

urlpatterns = [
    path("", include("home.urls")),
    path("accounts/", include("allauth.urls")),
    path("api/v1/", include("home.api.v1.urls")),
    path("admin/", admin.site.urls),
    path("users/", include("users.urls", namespace="users")),
    path("rest-auth/", include("rest_auth.urls")),
    # Override email confirm to use allauth's HTML view instead of rest_auth's API view
    path("rest-auth/registration/account-confirm-email/<str:key>/", confirm_email),
    path("rest-auth/registration/", include("rest_auth.registration.urls")),
]

admin.site.site_header = "Mineblox"
admin.site.site_title = "Mineblox Admin Portal"
admin.site.index_title = "Mineblox Admin"

# swagger
schema_view = get_schema_view(
    openapi.Info(
        title="Mineblox API",
        default_version="v1",
        description="API documentation for Mineblox App",
    ),
    public=True,
    permission_classes=(permissions.IsAuthenticated,),
)

urlpatterns += [
    path("api-docs/", schema_view.with_ui("swagger", cache_timeout=0), name="api_docs")
]
