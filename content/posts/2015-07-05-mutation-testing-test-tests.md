---
title: Mutation Testing or How to Test Tests
author: Ben E. Boyter
type: post
date: 2015-07-05T07:56:03+00:00
url: /2015/07/mutation-testing-test-tests/
nkweb_code_in_head:
  - default
nkweb_Use_Custom_js:
  - default
nkweb_Use_Custom_Values:
  - default
nkweb_Use_Custom:
  - 'false'
categories:
  - Testing
  - Unit Testing

---
Mutation testing is a technique used to verify that tests are providing value. Mutation testing involves modifying the given program in small ways. These could include changing boolean checks such as if a condition is True to being False. A mutated version of code is known as a mutant. For each mutant a the test suite is run against it. The tests when run over the mutant version should have a percentage of failure. Where a mutant is not caught additional tests can be written to cover these cases.

Mutation testing works well where you have a reasonable level of code coverage over your code and can be quite effective when done with test driven development. In my and others experience well written tests should have about 70% failure rate against code that has been mutated.

Quite a few mutation testing frameworks are out there such as Heckle, Insure++, Nester however you can get away with a simple find replace in most cases. Because mutation testing requires a lot of manual intervention to review the results it is usually something you would put into the code review process rather then as part of your continuous integration system. This is because even after mutating the code many of the tests may still pass. This might be due to the code not being conditional and still producing the correct output rather then ineffective tests.

I have written a very simple mutation tester which can be used against most languages and hosted it on github for your convenience <https://github.com/boyter/Mutator/>