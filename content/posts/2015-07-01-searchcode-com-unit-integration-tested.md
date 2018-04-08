---
title: How searchcode.com is Unit and Integration Tested
author: Ben E. Boyter
type: post
date: 2015-07-01T07:46:18+00:00
url: /2015/07/searchcode-com-unit-integration-tested/
nkweb_code_in_head:
  - default
nkweb_Use_Custom_js:
  - default
nkweb_Use_Custom_Values:
  - default
nkweb_Use_Custom:
  - 'false'
categories:
  - searchcode
  - Testing

---
searchcode is a source code and documentation search engine. It allows users to is search over 20 billions lines of code and millions of API documentation. It at the time of writing gets over 300,000 unique visitors a month processes many millions of API requests.

What follows is how I am testing searchcode, the issues I hit and where I think I am getting the most value. As big believer in testing as a methodology of improving code quality, one of my goals when rewriting searchcode was to ensure that it had a comprehensive test suite.

A brief overview of the architecture is required to understand what needs to be tested. At its core searchcode is a MySQL backed Django application using Sphinx as its indexing solution. The sphinx pipeline is the first thing that I wanted tested when I started the project. Because of the modifications to sphinx's indexing pipeline and what characters Sphinx itself became a candidate for testing. Not its code persay but how it is configured. The tests start with the Sphinx config file. Before each run it is copied from source control and configured to be used by Sphinx. A collection of terms which are known to have caused issues in the past is kept. During the test each term is indexed using sphinx then a series of queries positive and negative matching cases are run. Due to the high amount of integration these tests take a while to run and generally are only run before a doing a full deployment or if changing how the pipeline operates.

The next series of tests is achieved using Django's unit test framework. Not really pure unit tests as Django will spin up a temp database, which does however ensure that the database connections work along with permissions which saves testing these separately. Each test is written to be as unit as possible with multiple assertions. These are reasonably fast to run and are usually run in a test driven development manner as a result. The tests are split by models with all view tests existing in a single file. View tests are very simple and generally integration tests are prefered. The model tests are very detailed and attempt to get 100% branch coverage of all code paths.

There are a number of custom libraries, celery tasks and helper modules. Each of these contain their own seperate tests which can be run individually. However since they are actually quite fast they tend to get lumped together and run in batch. A lot of these tests run regressions which are taken from issues found on the main website. Rather then craft individual specific tests I have taken the approach of including the source code source for the test case. This ensures that any bizzare edge case in the logic itself is caught. Because of this there is a large collection of source code which does not belong to search code found with these tests.

A larger integration test suite is also run against a running Django instance. The instance is usually just the Django test webserver used for development however occasionally they are run against a full stack. The following steps are done for this. The first is that the parser used by searchcode to index code is run against searchcodes own source code. This is then indexed before any integration is run. The integration suite then using very simple urllib2 calls hits various URL's and checks the return values. These tests are favoured for things such as API calls as it mimics as closely as possible how the API would actually be used. The asserts are generally as simple as checking if a specific string is in the return and the return code. This avoids creating flaky tests that break with any style changes.

Other integration tests run against the small amount of Javascript running on the site. These are very limited and consist of running a PhantomJS headless browser against several URL's which should produce Ajax requests. The calls are checked to ensure that the requests were made and that there were no Javascript issues on the page. They act more as a sanity filter as while the request is checked the result is not, hence they do not generally catch changes to the API breaking things. Certainly there is room for improvement here.

Finally there is a collection of load tests. These are run very infrequently since the production rollout. They consist of a collection of URL's which are fed into siege. For the test a full production ready instance is spun up with a subset of production data. The siege test is run against this and at the conculustion the error logs and memcached stats are pulled back for analysis. This is mostly a manual process of checking to ensure that everything seems fine. Since the actual production deployment it has only been run when performing major caching changes which had the potential to cause large outages. The other collection of load tests consist of of running SQL which increases the database size to about 100,000 items before indexing. This quickly identifies any performance problems when large algorithm changes are made.

With the exception of load tests all tests are grouped into into fast and slow categories. The fast consist of the Django, library and full suite of integration tests. Generally they run in 20 seconds or less and are run frequently during development, and always before comitting changes. When work is performed on a specific modules the tests are run in isolation. Before any production deployment the full suite of tests are run which takes about two minutes to run. Each group of test belongs to a seperate fabic task and allows for quick checks over most of the codebase to ensure everything is working correctly before pushing changes live.

As would be expected the most value comes from the integration tests running against an active website. Pushing most code paths these tests cover all the functionality and most of the code paths. It is also quite lucky that searchcode is able to "self host" and run integration tests against its own code. This makes it a real test case and has proven to be one of the most valuable parts of the whole process.

The whole test quite was designed from the beginning to have no "flaky" tests. That is every test must run in the same manner every time it is run. Since integration tests are the usual suspect for flakey tests. purging the database and reloading it each time which slowed down the test suite but saved time tracking down issues with broken tests. There are also no external calls to 3rd parties which really helps ensure that the tests work each time.

I have found that these suites in practice have caught all issues before they went live. In fact the only time that any breaking bug was intruduced was due to myself skipping parts of the process and pushing live before running the full spectrum of tests. Generally most issues are due to being unable to have a local copy of production data due to size constraints.

Any questions or comments? Feel free to email me at ben@boyter.org or hit me up on twitter @boyter