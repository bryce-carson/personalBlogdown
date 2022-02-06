---
title: 'Packaging SLiM for Debian and Ubuntu: a difficult process'
author: 'Bryce Carson'
date: '2022-02-05'
slug: []
categories: ["Population Genetics", "Genetics", "Programming"]
tags: ["Debian", "Ubuntu", "Packaging", "Development"]
ShowToC: true
editPost:
  URL: "https://github.com/bryce-carson/personalBlogdown/content/tree/main"
  Text: "Suggest changes to this post"
  appendFilePath: true
---

# Previous Experience Packaging
I packaged [SLiM for Fedora, CentOS, Red Hat Enterprise, and
openSUSE](https://copr.fedorainfracloud.org/coprs/bacarson/SLiM-Selection_on_Linked_Mutations/)
over a year ago. Learning how to package software in the RPM format did take
some time, but it was not too difficult after I read the excellent documentation
and did some internet searches for various macros where the documentation did
not specify (because it had not been updated, or the macro was
distribution-specific and I would've needed to consult that specific
documentation).

-------------------------------------------------------------------------------

# *SLiM: Selection on Linked Mutations* Simulation Framework
![SLiMgui Screenshot](img/slimGUI-screenshot.png)

# Debian Packaging
## Ubuntu Packaging
