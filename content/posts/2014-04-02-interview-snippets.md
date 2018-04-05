---
title: More interview snippets….
author: Ben E. Boyter
type: post
date: 2014-04-02T08:59:36+00:00
url: /2014/04/interview-snippets/
categories:
  - Interviews

---
Since I wrote the code to these snippets I thought I may as well add them here in case I ever need them again or want to review them. As the other interview ones they are the answers to a question I was asked, slightly modified to protect the innocent. These ones are written in Python.

Q. Write a function to reverse each word in a string.

<pre>def reverse_each_word(words):
    '''
    Reverse each word in a string 
    '''
    return " ".join([x[::-1] for x in words.split(' ')])</pre>

The only thing of note in here is the x[::-1] which is extended slice syntax which reverses a string. You could also to reversed(x) although I believe at the time of writing it is MUCH slower.

Q. Given two arrays find which elements are not in the second.

<pre>def find_not_in_second(first, second): 
    '''
    Find which numbers are not in the
    second array
    '''
    return [x for x in first if x not in second]</pre>

I am especially proud of the second snippet as it is very easy to read and rather Pythonic. It takes in two lists such as [1,2,3] and [2,3,6] and returns a new list with the missing elements.