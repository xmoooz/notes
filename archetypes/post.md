---
author: xmoooz
categories: General
comments: true
date: {{ .Date }}
lastmod: {{ .Date }}
draft: true
tags: []
title: "{{ replace (strings.TrimPrefix (printf "%s-" (substr .Date 0 10)) .File.ContentBaseName) "-" " " | title }}"
---
