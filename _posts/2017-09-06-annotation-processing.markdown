---
layout: post
title:  "Annotation Processing"
date:   2017-09-06 12:02:49 +0300
categories: annotations
---
Annotation Processing
======
I won't describe how is it works because Hannes Dorfmann has already done it here - [Annotation Processing 101](http://hannesdorfmann.com/annotation-processing/annotationprocessing101)

But he didn't describe how to debug annotation processing. So I recommend you to look at this - [How do you debug Java annotation processors using IntelliJ?](https://stackoverflow.com/a/42488641/1534522)

Keep in mind when using `Kotlin` and annotates properties `Kotlin` automatically generates `getters` and `setters` so in generated files you would have to call them instead. If you want have `public` field in `Kotlin` just add one more annotation - `@JvmField` 

Moreover, when you use `Kotlin` do not forget to use `kapt` instead of `apt`.