.SILENT:
.ONESHELL:
SHELL := bash
.SHELLFLAGS := -eu -o pipefail -c

QW_LAMBDA_VERSION?=$(shell git tag --sort=committerdate | grep -E "aws-lambda-beta-" | tail -1 | cut -b 12-)
$(if $(QW_LAMBDA_VERSION),,$(error "Version tag not found, try 'git fetch --tags' or specify QW_LAMBDA_VERSION=beta-<version>"))
PACKAGE_BASE_URL=https://github.com/quickwit-oss/quickwit/releases/download/aws-lambda-$(QW_LAMBDA_VERSION)/
SEARCHER_PACKAGE_FILE=quickwit-lambda-searcher-$(QW_LAMBDA_VERSION)-x86_64.zip
INDEXER_PACKAGE_FILE=quickwit-lambda-indexer-$(QW_LAMBDA_VERSION)-x86_64.zip
export SEARCHER_PACKAGE_PATH=cdk.out/$(SEARCHER_PACKAGE_FILE)
export INDEXER_PACKAGE_PATH=cdk.out/$(INDEXER_PACKAGE_FILE)

check-env:
ifndef CDK_ACCOUNT
	$(error CDK_ACCOUNT is undefined)
endif
ifndef CDK_REGION
	$(error CDK_REGION is undefined)
endif

# Build or download the packages from the release page
# - Download by default, the version can be set with QW_LAMBDA_VERSION
# - To build locally, set QW_LAMBDA_BUILD=1
package:
	mkdir -p cdk.out
	if [ "$${QW_LAMBDA_BUILD:-0}" = "1" ]
	then
	  pushd ../../quickwit/
	  rustc --version
	  # TODO: remove --disable-optimizations when upgrading to a release containing
	  # https://github.com/cargo-lambda/cargo-lambda/issues/649 (> 1.2.1)
	  cargo lambda build \
	    -p quickwit-lambda \
		--disable-optimizations \
	    --release \
	    --output-format zip \
	    --target x86_64-unknown-linux-gnu
	  popd
	  cp -u ../../quickwit/target/lambda/searcher/bootstrap.zip $(SEARCHER_PACKAGE_PATH)
	  cp -u ../../quickwit/target/lambda/indexer/bootstrap.zip $(INDEXER_PACKAGE_PATH)
	else
	  if ! [ -f $(SEARCHER_PACKAGE_PATH) ]; then
	  	echo "Downloading package $(PACKAGE_BASE_URL)$(SEARCHER_PACKAGE_FILE)"
	    curl --fail -C - -Ls -o $(SEARCHER_PACKAGE_PATH) $(PACKAGE_BASE_URL)$(SEARCHER_PACKAGE_FILE)
	  else
	  	echo "Using cached package $(SEARCHER_PACKAGE_PATH)"
	  fi
	  if ! [ -f $(INDEXER_PACKAGE_PATH) ]; then
	  	echo "Downloading package $(PACKAGE_BASE_URL)$(INDEXER_PACKAGE_FILE)"
	  	curl --fail -C - -Ls -o $(INDEXER_PACKAGE_PATH) $(PACKAGE_BASE_URL)$(INDEXER_PACKAGE_FILE)
	  else
	  	echo "Using cached package $(INDEXER_PACKAGE_PATH)"
	  fi
	fi

indexer-package-path:
	echo -n $(INDEXER_PACKAGE_PATH)

searcher-package-path:
	echo -n $(SEARCHER_PACKAGE_PATH)

bootstrap:
	cdk bootstrap aws://$$CDK_ACCOUNT/$$CDK_REGION

deploy-hdfs: package check-env
	cdk deploy --require-approval never -a cdk/app.py HdfsStack

deploy-mock-data: package check-env
	cdk deploy  --require-approval never -a cdk/app.py MockDataStack

print-mock-data-metastore: check-env
	python -c 'from cdk import cli; cli.print_mock_data_metastore()'

# address https://github.com/aws/aws-cdk/issues/20060
before-destroy:
	mkdir -p cdk.out
	touch $(INDEXER_PACKAGE_PATH)
	touch $(SEARCHER_PACKAGE_PATH)

destroy-hdfs: before-destroy check-env
	python -c 'from cdk import cli; cli.empty_hdfs_bucket()'
	cdk destroy --force -a cdk/app.py HdfsStack

destroy-mock-data: before-destroy check-env
	python -c 'from cdk import cli; cli.empty_mock_data_buckets()'
	cdk destroy --force -a cdk/app.py MockDataStack

clean:
	rm -rf cdk.out

## Invocation examples

invoke-mock-data-searcher: check-env
	python -c 'from cdk import cli; cli.invoke_mock_data_searcher()'

invoke-hdfs-indexer: check-env
	python -c 'from cdk import cli; cli.upload_hdfs_src_file()'
	python -c 'from cdk import cli; cli.invoke_hdfs_indexer()'

invoke-hdfs-searcher-term: check-env
	python -c 'from cdk import cli; cli.invoke_hdfs_searcher("""{"query": "severity_text:ERROR", "max_hits": 10}""")'

invoke-hdfs-searcher-histogram: check-env
	python -c 'from cdk import cli; cli.invoke_hdfs_searcher("""{ "query": "*", "max_hits": 0, "aggs": { "events": { "date_histogram": { "field": "timestamp", "fixed_interval": "1d" }, "aggs": { "log_level": { "terms": { "size": 10, "field": "severity_text", "order": { "_count": "desc" } } } } } } }""")'

bench-index:
	mem_sizes=( 10240 8192 6144 4096 3072 2048 )
	export QW_LAMBDA_DISABLE_MERGE=true
	for mem_size in "$${mem_sizes[@]}"
	do
		export INDEXER_MEMORY_SIZE=$${mem_size}
		$(MAKE) deploy-hdfs
		python -c 'from cdk import cli; cli.benchmark_hdfs_indexing()'
	done

bench-search-term:
	export QW_LAMBDA_ENABLE_VERBOSE_JSON_LOGS=true
	mem_sizes=( 1024 2048 4096 8192 )
	for mem_size in "$${mem_sizes[@]}"
	do
		export SEARCHER_MEMORY_SIZE=$${mem_size}
		$(MAKE) deploy-hdfs
		python -c 'from cdk import cli; cli.benchmark_hdfs_search("""{"query": "severity_text:ERROR", "max_hits": 10}""")'
	done

bench-search-histogram:
	export QW_LAMBDA_ENABLE_VERBOSE_JSON_LOGS=true
	mem_sizes=( 1024 2048 4096 8192 )
	for mem_size in "$${mem_sizes[@]}"
	do
		export SEARCHER_MEMORY_SIZE=$${mem_size}
		$(MAKE) deploy-hdfs
		python -c 'from cdk import cli; cli.benchmark_hdfs_search("""{ "query": "*", "max_hits": 0, "aggs": { "events": { "date_histogram": { "field": "timestamp", "fixed_interval": "1d" }, "aggs": { "log_level": { "terms": { "size": 10, "field": "severity_text", "order": { "_count": "desc" } } } } } } }""")'
	done

bench-search:
	for run in {1..30}
	do 
		export QW_LAMBDA_PARTIAL_REQUEST_CACHE_CAPACITY=0
		$(MAKE) bench-search-term
		$(MAKE) bench-search-histogram
		export QW_LAMBDA_PARTIAL_REQUEST_CACHE_CAPACITY=64MB
		$(MAKE) bench-search-term
		$(MAKE) bench-search-histogram
	done

test-mock-data-endpoints:
	python -c 'from cdk import cli; cli.test_mock_data_endpoints()'
