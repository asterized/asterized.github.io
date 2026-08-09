---
layout: post
title:  "PyPI package downloads"
date:   2026-08-09 10:40:19
categories: jekyll update
katex: false
---

I happened to publish a package recently ([check it out if you wish](https://pypi.org/project/voprf/)), and because I like statistics, I decided to check the download statistics of the package a bit later through [PyPI Stats](https://pypistats.org), and saw:

![Download quantity of a package. Download numbers stayed extremely low (<15 except initial release date) until 2026-08-07, where downloads peaked to 3,411 and went down to 1,673 the day after](/assets/images/stats-1.png)

Woah, that's weird. Why would that be?

## Potential explanations

### Potential explanation 1: It blew up and all of it is humans

Yeah, no. I hadn't shared it anywhere really and the package usecase is niche enough that not that many people will care.

### Potential explanation 2: All of it is security scanners

PyPI has a large number of scanners that scan the entire PyPI registry for malware, such as [Vipyr Security](https://vipyrsec.com). These scanners go through packages, checking their source code against rules to ensure that malware (mostly) stays off PyPI. During 2026-08-07, I uploaded a new version to the package, which would likely trigger the scanners, and the naming is similar and the tags might cause increased attention on the package, so this immediately showed up as the most plausible explanation.

This idea also checks out as the vast majority of these downloads did not come from package managers such as `pip`, but directly through a HTTP request to the PyPI registry, as for most downloads, the Python version it was downloaded to and the operating system were not known.

However, checking similar packages on PyPI Stats, none of them had the download numbers that high, and the "base" level of downloads before -08-07 were relatively low and likely from similar malware scanners, and asking people in the Vipyrsec team (thanks guys) told me the package was only scanned once, and although this doesn't represent *every* scanner, there surely arent more than 500 malware scanners checking PyPI 24/7, and if there were these patterns weren't seen from packages with new versions near the same timetold me the package was only scanned once, and although this doesn't represent *every* scanner, there surely arent more than 500 malware scanners checking PyPI 24/7, and if there were these patterns weren't seen from packages with new versions near the same time.

### Potential explanation 3: AI scrapers

This seemed like the most plausible explanation, as they might not be solely targeted towards PyPI (hence not seeing the same patterns from other packages). Furthermore, it's known that they [aren't](https://anubis.techaro.lol/) [particularly](https://drewdevault.com/blog/Stop-externalizing-your-costs-on-me/) [efficient](https://www.phoronix.com/news/GNOME-GitLab-Fastly). This explanation essentially assumes that multiple AI scrapers went to hit the package likely multiple times, which seems like a plausible explanation.

## The real (?) answer

Of course, this is still a "best guess" answer, but its almost definitely the correct answer.

So, why was this sudden spike? I wasn't really seeing any other explanation, and none of the three seemed satisfying to me, until someone else said:

> Ah, it's the number of wheels you built amplifying your download count lol.

... oh.

### Wheels

PyPI bundles packages in "wheels", which is essentially a fancy zip file. Usually, pure Python packages are distributed in a single wheel, but wheels were made for Python packages with C extensions, and mine is written mainly in Rust (which interfaces using the same ABI as C extensions).

I chose to use maturin's [GitHub Action](https://github.com/PyO3/maturin-action) to package it, which allows wheels built for multiple operating systems and architectures, with my package building about 101 wheels, for every permutation of interpreter (CPython and PyPy), ABI (because they change a bunch, from Python 3.9 to 3.15t, and the PyPy ABI), and platform-OS target (manylinux/musllinux/win/macOS, and every permutation of platforms such as x86, x86_64, ARM, ...).

### So...?

As it turns out, malware scanners went and scanned through every wheel, overinflating download statistics. In reality, there were only about 34 scans going through the package. This possibly also explains why there were over 200 installs from Python 3.10 (the second highest Python version which installed it, behind `Null`/unknown). Python 3.10 is the oldest supported version of CPython (which possibly means the most bug patches). These downloads were likely [dynamic analysis](https://en.wikipedia.org/wiki/Dynamic_program_analysis) tools going through and scanning the wheels.

## Conclusion

Thanks to people from the Vipyrsec team for explaining why I got everything wrong, and now I can sleep at night knowing no one relies on my software.
