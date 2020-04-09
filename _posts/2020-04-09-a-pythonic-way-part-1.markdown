---
layout: post
title:  "Python - A Pythonic Way (Part I)"
date:   2020-04-09 18:30:00 +0700
categories: [python, pythonic, best_practices]
---
**Python** - one of the world's most popular programming language of 2019 according to Stackoverflow research. It’s one really powerful language which provides you many tools to make your life (and your team life) easier. But like a coin with two sides, if you know how how to use those tools effectively, then, everything will be great, else, you will not gain much of the benefit from this trending language.


So from this series, I want to help you understand what is the Pythonic way of code that everyone is talking about. And yet, you mind think “Yeah, another python best practices article” but in this, I want to structure everything in a better way. Also kind of a note for myself in case of my goldfish memory “accidentally” forgot this, I could have a reference too ! 


## Timeline
Let start, the structure of this series will contain the following contents:  
- Environment 
- Code  
	- Code syntax, project structure, module packaging
	- Build in functions, data structures, modules
- Documentation
- Test - Unittest


## Enviroment
### Python version
From January 1st 2020, **python 2** is officially **End of Life**, out of support from the community for bug fixes, improvements. So in another word, if you are using python 2, migrate your system to python 3 is a must. If you starting a new project, **use python 3** for it you will have better support and for sure many more new features will be in there to help you do your job faster.

### Virtual environment
The first rule when working with python is always, **always** create a **new python environment** for each project you are working on. And in here, I would recommend using [Anaconda](https://www.anaconda.com/) for this since it can handle multiple python version environment separated, also, it has supported a lot of C++ libraries underneath it system which can save you a lot of time when working on a project that requires package like numpy, scipy, sklearn, tensorflow,... basically, if you working as a data scientist, conda is a must.

But besides the support for C++ lib, why we need to separate python environment? We can still install C++ lib in by ourself, system-wide installation, one-time job for all project? 

The answer is the **dependency graph**. Each project you are working on will require some set of packages. Like other programming languages, when you install a package, that package will require (depend) on some other package and so on, the dependency chain keeps going. Until you have 2 packages A in project A', B in project B' that requires the same package C, and however, A require version 1.19 of C, B requires version 2.11 of C. Now things are about getting messy, which version of package C should be installing? One strategy is install the newer version - 2.11, the best one we can find. But then how we can tell if package A will run correctly as we want? The problem above is just one of many problem dependency graphs have, you can read more about this in [this wiki page](https://en.wikipedia.org/wiki/Dependency_hell) about **Dependency Hell** - you can see, the name has spoken for itself it's hell down that road !

From my experience, we have many painful and yet, time-consuming issue to debug with different package versions lead to. So, create a new environment each time.

Another reason for this can be, it'll be easier for you to "clone" other person's development environment in the fastest way. You can copy the entire conda environment from one machine to another one, it'll work just fine. No system path change, no extra code needed.


## Code
### Syntax
Python community comes with a long list of proposals for enhancement for python language which in short is PEP. More detail can be found [Python official website](https://www.python.org/dev/peps/). For syntax, python community recommends us to follow the guideline in PEP8 - Style Guide for Python Code. You should always **enforce** this rule to your CI/CD pipeline or code review process for **every project**.

Follow this guideline will help us:
- The code syntax consistent between projects. 
- Maintain the readability of python code.
- Avoid some weird behavior of different syntax, code structure or common mistake.

So we need to read and remember like grade 1 student all this f***ing long guide to code? The answer is NO, you don't. Just read it one time and forget about it. Install flake8 package and it'll do the check if you violate any rule for you. Flake8 is a wrapper for 3 tools:
- pyflakes: a package that analyzes programs and detects various errors (one common case can be: find the undefined variable but still being use). It works by parsing the source file, not importing it, so it is safe to use on modules with side effects.
- pycodestyle: a tool to check your Python code against some of the style conventions in PEP 8.
- Ned Batchelder's McCabe script: this package allow you to write and add your custom syntax check plugin module to flake8 check process.

Here is an example for flake8 in action. I have this python snippet save as _test_pep8.py_.
```python
for i in range(10):
    int_list.append(i)
    import time
    time.sleep(1)
	print(f"Counting to number {i}")
```
And then I can run flake8 with the command line and then I have is a list of the problem that I should be looking at.
```
(py36) C:\Users\nguye>flake8 test_pep8.py
test_pep8.py:5:1: W191 indentation contains tabs
test_pep8.py:5:1: E101 indentation contains mixed spaces and tabs
test_pep8.py:5:2: E113 unexpected indentation
test_pep8.py:5:33: E999 TabError: inconsistent use of tabs and spaces in indentation
test_pep8.py:5:34: W292 no newline at end of file
```
After that, we fix it and all good.
```python
import time


int_list = []
for i in range(10):
    int_list.append(i)
    time.sleep(1)
    print(f"Counting to number {i}")

```
So just give yourself some time to be become familiar with python syntax, after a couple (maybe more) time of making mistake, you will master it.


# Closing
I think that is the end of the first part of this series, see you in the next part.