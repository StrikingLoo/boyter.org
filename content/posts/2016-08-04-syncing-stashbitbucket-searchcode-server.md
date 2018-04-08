---
title: Syncing Stash/BitBucket with searchcode server
author: Ben E. Boyter
type: post
date: 2016-08-04T05:14:19+00:00
url: /2016/08/syncing-stashbitbucket-searchcode-server/
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

---
Recently it came up to perform a slight integration piece between a on premises Stash/BitBucket install and a searchcode server install. Thankfully both have an API and very thankfully there is a nice Python library for talking to Stash/BitBucket.

Below is the code used. It pulls out all of the repositories from every project, checks if it exists in searchcode and if not adds it as a repository to be indexed. You need to install stashy (pip install stashy) and run it whenever you have new repositories. One idea is to set it as a cron task and ensure everything is in sync.

Note that this does not remove repositories that have been indexed, but it would not take much work to achieve it.

{{<highlight python>}}
import stashy
from hashlib import sha1
from hmac import new as hmac
import urllib2
import json
import urllib

def getstashrepos():
    stash = stashy.connect("https://mystashserver/", "STASH_USERNAME", "STASH_PASSWORD")

    projects = stash.projects.list()
    repos = [stash.projects[x['key']].repos.list() for x in projects]

    stashrepos = []

    for repo in repos:
        stashrepos = stashrepos + [{'name': x['project']['key'] + '-' + x['slug'],
                                    'cloneUrl': x['cloneUrl'],
                                    'browse': x['links']['self'][0]['href']} for x in repo]

    return stashrepos

def addtosearchcode(repo):
    reponame = repo['name']
    repourl = repo['cloneUrl']
    repotype = "git"
    repousername = "STASH_USERNAME"
    repopassword = "STASH_PASSWORD"
    reposource = repo['browse']
    repobranch = "master"

    message = "pub=%s&reponame=%s&repourl=%s&repotype=%s&repousername=%s&repopassword=%s&reposource=%s&repobranch=%s" % (
            urllib.quote_plus(publickey),
            urllib.quote_plus(reponame),
            urllib.quote_plus(repourl),
            urllib.quote_plus(repotype),
            urllib.quote_plus(repousername),
            urllib.quote_plus(repopassword),
            urllib.quote_plus(reposource),
            urllib.quote_plus(repobranch)
        )

    sig = hmac(privatekey, message, sha1).hexdigest()

    url = "http://mysearchcodeserver/api/repo/add/?sig=%s&%s" % (urllib.quote_plus(sig), message)

    data = urllib2.urlopen(url)
    data = data.read()

    data = json.loads(data)
    print reponame, data['sucessful'], data['message']


publickey = "MY_SEARCHCODE_PUBLIC_KEY"
privatekey = "MY_SEARCHCODE_PRIVATE_KEY"

message = "pub=%s" % (urllib.quote_plus(publickey))

sig = hmac(privatekey, message, sha1).hexdigest()
url = "http://mysearchcodeserver/api/repo/list/?sig=%s&%s" % (urllib.quote_plus(sig), message)

data = urllib2.urlopen(url)
data = data.read()

data = json.loads(data)
existingrepos = [x['name'] for x in data['repoResultList']]

for repo in getstashrepos():
    if repo['name'] not in existingrepos:
        addtosearchcode(repo)
{{</highlight>}}