---
title: "Gradle初始化配置"
date: 2022-03-06T11:22:39+08:00
draft: true
tags: ["java", "gradle"]
toc: false
---

我的init.gradle配置

```groovy
def aliyunMavenRepoUrl = 'http://maven.aliyun.com/nexus/content/groups/public'

allprojects {
    repositories {
        add mavenLocal()
        add maven {
            allowInsecureProtocol = true
            name "aliyunMaven"
            url aliyunMavenRepoUrl
        }
    }

    repositories {
        var repoUrls = []
        all {
            ArtifactRepository repo ->
                {
                    if (repo instanceof MavenArtifactRepository) {
                        repoUrls.add(repo.url.toString())
                    }
                }
        }
        var repoUrlsJoin = repoUrls.join("; ")
        project.logger.lifecycle "project ${project.name} insert maven repos: ${repoUrlsJoin}"
    }
}
```
