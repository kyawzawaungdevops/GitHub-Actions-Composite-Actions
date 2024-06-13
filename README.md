# Reusable Laravel CI Workflow

This repository demonstrates a reusable GitHub Actions workflow for building and testing a Laravel application. Below is the YAML configuration for the reusable workflow (`reusable-workflow.yaml`) and how it can be called from another workflow (`caller-workflow.yaml`).

```yaml
# Reusable Workflow Definition (reusable-workflow.yaml)
name: Reusable Workflow

on:
  workflow_call:

jobs:
  build:
    name: Build Laravel App
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: "8.3"

      - name: Install dependencies
        run: |
          composer install

      - name: Cache the Vendor
        uses: actions/cache@v4.0.2
        with:
          key: ${{ github.sha }}-php-vendor-cache
          path: ./vendor

      - name: Create .env
        run: |
          cp .env.ci .env
          cp .env.ci .env.testing

      - name: Generate the key
        run: |
          php artisan key:generate
        env:
          APP_KEY: $(php artisan key:generate --show)

  test:
    name: Test the Application
    runs-on: ubuntu-latest
    needs: [build]
    services:
      mysql:
        image: mysql:latest
        ports:
          - 3306:3306
        env:
          MYSQL_ALLOW_EMPTY_PASSWORD: false
          MYSQL_ROOT_PASSWORD: password
          MYSQL_USERNAME: root
          MYSQL_DATABASE: test
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: "8.3"

      - name: Restore the Cache
        id: cache
        uses: actions/cache@v4.0.2
        with:
          path: ./vendor
          key: ${{ github.sha }}-php-vendor-cache

      - name: Composer Install
        if: steps.cache.outputs.cache-hit != 'true'
        run: |
          composer install

      - name: Create .env
        run: |
          cp .env.ci .env
          cp .env.ci .env.testing

      - name: Generate the key
        run: |
          php artisan key:generate --show > key.txt
          APP_KEY=$(cat key.txt)
          sed -i "s|^APP_KEY=.*|APP_KEY=${APP_KEY}|" .env

      - name: Migrate the Database
        run: |
          php artisan migrate

      - name: Run Unit Testing
        run: |
          php artisan test

# Caller Workflow (caller-workflow.yaml)
on:
  push:
  pull_request:
    branches:
      - main
      - dev

jobs:
  build-and-test:
    uses: ./.github/workflows/reusable-workflow.yaml
