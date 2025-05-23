.DEFAULT_GOAL := help

.PHONY: help clean clean_tox docs requirements ci_requirements dev_requirements \
        validation_requirements doc_requirements production-requirements static shell \
        test coverage isort_check isort style lint quality pii_check validate \
        migrate html_coverage upgrade compile-requirements extract_translation dummy_translations \
        compile_translations fake_translations pull_translations \
        start-devstack open-devstack pkg-devstack \
        detect_changed_source_translations validate_translations check_keywords \
        install_transifex_client

# For opening files in a browser. Use like: $(BROWSER)relative/path/to/file.html
BROWSER := python -m webbrowser file://$(CURDIR)/

# Generates a help message. Borrowed from https://github.com/pydanny/cookiecutter-djangopackage.
help: ## display this help message
	@echo "Please use \`make <target>\` where <target> is one of"
	@awk -F ':.*?## ' '/^[a-zA-Z]/ && NF==2 {printf "\033[36m  %-25s\033[0m %s\n", $$1, $$2}' $(MAKEFILE_LIST) | sort

clean: ## delete generated byte code and coverage reports
	find . -name '*.pyc' -delete
	coverage erase
	rm -rf assets
	rm -rf pii_report

clean_tox: ## clear tox requirements cache
	rm -fr .tox

docs: ## generate Sphinx HTML documentation, including API docs
	tox -e docs
	$(BROWSER)docs/_build/html/index.html

piptools: ## install pinned version of pip-compile and pip-sync
	pip install -r requirements/pip.txt
	pip install -r requirements/pip-tools.txt

requirements: dev_requirements ## sync to default requirements

ci_requirements: validation_requirements ## sync to requirements needed for CI checks

dev_requirements: clean_tox piptools ## sync to requirements for local development
	pip-sync -q requirements/dev.txt requirements/private.*

validation_requirements: piptools ## sync to requirements for testing & code quality checking
	pip-sync -q requirements/validation.txt

doc_requirements: piptools
	pip-sync -q requirements/doc.txt

production-requirements: piptools ## install requirements for production
	pip-sync -q requirements/production.txt

static: ## generate static files
	python manage.py collectstatic --noinput

shell: ## run Django shell
	python manage.py shell

test: clean ## run tests and generate coverage report
	pytest

# To be run from CI context
coverage: clean
	pytest --cov-report html
	$(BROWSER)htmlcov/index.html

isort_check: ## check that isort has been run
	isort --check-only {{cookiecutter.project_name}}/

isort: ## run isort to sort imports in all Python files
	isort --atomic {{cookiecutter.project_name}}/

style: ## run Python style checker
	pylint --rcfile=pylintrc {{cookiecutter.project_name}} *.py

lint: ## run Python code linting
	pylint --rcfile=pylintrc {{cookiecutter.project_name}} *.py

quality:
	tox -e quality

pii_check: ## check for PII annotations on all Django models
	DJANGO_SETTINGS_MODULE={{cookiecutter.project_name}}.settings.test \
	code_annotations django_find_annotations --config_file .pii_annotations.yml --lint --report --coverage

check_keywords: ## Scan the Django models in all installed apps in this project for restricted field names
	python manage.py check_reserved_keywords --override_file db_keyword_overrides.yml

validate: test quality pii_check check_keywords ## run tests, quality, and PII annotation checks

migrate: ## apply database migrations
	python manage.py migrate

html_coverage: ## generate and view HTML coverage report
	coverage html && open htmlcov/index.html

# Define PIP_COMPILE_OPTS=-v to get more information during make upgrade.
PIP_COMPILE = pip-compile $(PIP_COMPILE_OPTS)

compile-requirements: ## compile the requirements/*.txt files with the latest packages satisfying requirements/*.in
	pip install -qr requirements/pip-tools.txt
	pip-compile -v ${COMPILE_OPTS} --allow-unsafe --rebuild -o requirements/pip.txt requirements/pip.in
	pip-compile -v ${COMPILE_OPTS} -o requirements/pip-tools.txt requirements/pip-tools.in
	pip install -qr requirements/pip.txt
	pip install -qr requirements/pip-tools.txt
	$(PIP_COMPILE) -o requirements/base.txt requirements/base.in
	$(PIP_COMPILE) -o requirements/test.txt requirements/test.in
	$(PIP_COMPILE) -o requirements/doc.txt requirements/doc.in
	$(PIP_COMPILE) -o requirements/quality.txt requirements/quality.in
	$(PIP_COMPILE) -o requirements/validation.txt requirements/validation.in
	$(PIP_COMPILE) -o requirements/ci.txt requirements/ci.in
	$(PIP_COMPILE) -o requirements/dev.txt requirements/dev.in
	$(PIP_COMPILE) -o requirements/production.txt requirements/production.in
	# Let tox control the Django version for tests
	grep -e "^django==" requirements/base.txt > requirements/django.txt
	sed '/^[dD]jango==/d' requirements/test.txt > requirements/test.tmp
	mv requirements/test.tmp requirements/test.txt

upgrade: ## update the requirements/*.txt files with the latest packages satisfying requirements/*.in
	pip install -qr requirements/pip-tools.txt
	$(MAKE) compile-requirements COMPILE_OPTS="--upgrade"

extract_translations: ## extract strings to be translated, outputting .mo files
	cd {{cookiecutter.project_name}} && i18n_tool extract --no-segment

dummy_translations: ## generate dummy translation (.po) files
	cd {{cookiecutter.project_name}} && i18n_tool dummy

compile_translations: # compile translation files, outputting .po files for each supported language
	python manage.py compilemessages

fake_translations: extract_translations dummy_translations compile_translations ## generate and compile dummy translation files

ifeq ($(OPENEDX_ATLAS_PULL),)
pull_translations: ## pull translations from Transifex
	tx pull -af -t --mode reviewed
else
# Experimental: OEP-58 Pulls translations using atlas
pull_translations:
	find {{cookiecutter.project_name}}/conf/locale -mindepth 1 -maxdepth 1 -type d -exec rm -r {} \;
	atlas pull $(OPENEDX_ATLAS_ARGS) translations/{{cookiecutter.repo_name}}/{{cookiecutter.project_name}}/conf/locale:{{cookiecutter.project_name}}/conf/locale
	python manage.py compilemessages

	@echo "Translations have been pulled via Atlas and compiled."
endif

start-devstack: ## run a local development copy of the server
	docker-compose --x-networking up

open-devstack: ## open a shell on the server started by start-devstack
	docker exec -it {{cookiecutter.repo_name}} /edx/app/{{cookiecutter.repo_name}}/devstack.sh open

pkg-devstack: ## build the {{cookiecutter.repo_name}} image from the latest configuration and code
	docker build -t {{cookiecutter.repo_name}}:latest -f docker/build/{{cookiecutter.repo_name}}/Dockerfile git://github.com/openedx/configuration

detect_changed_source_translations: ## check if translation files are up-to-date
	cd {{cookiecutter.project_name}} && i18n_tool changed

validate_translations: fake_translations detect_changed_source_translations ## install fake translations and check if translation files are up-to-date

docker_build:
	docker build . -f Dockerfile -t openedx/{{cookiecutter.repo_name}}

# devstack-themed shortcuts
dev.up: # Starts all containers
	docker-compose up -d

dev.up.build:
	docker-compose up -d --build

dev.down: # Kills containers and all of their data that isn't in volumes
	docker-compose down

dev.stop: # Stops containers so they can be restarted
	docker-compose stop

app-shell: # Run the app shell as root
	docker exec -u 0 -it {{cookiecutter.project_name}}.app bash

db-shell: # Run the app shell as root, enter the app's database
	docker exec -u 0 -it {{cookiecutter.project_name}}.db mysql -u root {{cookiecutter.project_name}}

%-logs: # View the logs of the specified service container
	docker-compose logs -f --tail=500 $*

%-restart: # Restart the specified service container
	docker-compose restart $*

%-attach:
	docker attach {{cookiecutter.project_name}}.$*

docker_auth:
	echo "$$DOCKERHUB_PASSWORD" | docker login -u "$$DOCKERHUB_USERNAME" --password-stdin

docker_tag_build_push: docker_auth
	docker buildx build -t openedx/{{cookiecutter.repo_name}}:latest -t openedx/{{cookiecutter.repo_name}}:${GITHUB_SHA} --platform linux/amd64,linux/arm64 --target prod --push .

docker_tag_build_push_dev: docker_auth
	docker buildx build -t openedx/{{cookiecutter.repo_name}}-dev:latest -t openedx/{{cookiecutter.repo_name}}-dev:${GITHUB_SHA} --platform linux/amd64,linux/arm64 --target dev --push .

selfcheck: ## check that the Makefile is well-formed
	@echo "The Makefile is well-formed."

install_transifex_client: ## Install the Transifex client
	# Installing client will skip CHANGELOG and LICENSE files from git changes
	# so remind the user to commit the change first before installing client.
	git diff -s --exit-code HEAD || { echo "Please commit changes first."; exit 1; }
	curl -o- https://raw.githubusercontent.com/transifex/cli/master/install.sh | bash
	git checkout -- LICENSE README.md ## overwritten by Transifex installer
