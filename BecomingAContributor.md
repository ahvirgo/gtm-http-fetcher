# How to become a contributor and submit patches #

## Contributor License Agreements ##

We'd love to accept your code patches. However, before we can take them, we have to jump through a couple of legal hurdles.

Please fill out either the individual or corporate Contributor License Agreement.

  * If you are an individual writing original source code and you're sure you own the intellectual property, then you'll need to sign an [individual CLA](http://code.google.com/legal/individual-cla-v1.0.html).
  * If you work for a company that wants to allow you to contribute your work to Google Toolbox for Mac, then you'll need to sign a [corporate CLA](http://code.google.com/legal/corporate-cla-v1.0.html).

Follow either of the two links above to access the appropriate CLA and instructions for how to sign and return it. Once we receive it, we'll add you to the official list of contributors and be able to accept your patches.

## Submitting Patches ##

  1. **FIXME** Join the Google Toolbox for Mac [discussion group](http://groups.google.com/group/google-toolbox-for-mac/).
  1. Decide which code you want to submit. A submission should be a set of changes that addresses one issue in the [issue tracker](http://code.google.com/p/gtm-http-fetcher/issues/list). Please don't mix more than one logical change per submittal, because it makes the history hard to follow. If you want to make a change that doesn't have a corresponding issue in the issue tracker, please create one.
  1. Also, coordinate with team members that are listed on the issue in question. This ensures that work isn't being duplicated. Communicating your plan early also generally leads to better patches.
  1. Ensure that your code adheres to the library source code style ([Objective C Style Guide](http://google-styleguide.googlecode.com/svn/trunk/objcguide.xml)).
  1. Sign a Contributor License Agreement.
  1. There are two ways to provide the code for review:
    * The _preferred_ way is to provide the code is with [Rietveld](http://codereview.appspot.com/), then simply append a comment to the issue your change addresses with a link to the review on Rietveld.  This lets anyone look at the changes in content and provides a system for comments on the change. `upload.py` ([here](http://codereview.appspot.com/static/upload.py)) automates the uploading from a svn tree.
    * If you'd prefer, you can attach the code to the issue it addresses directly. For brand new files, attach the whole file. For patches to existing files, attach a Subversion diff (`svn diff`).