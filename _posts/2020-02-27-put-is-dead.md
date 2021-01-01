---
layout: post
title: HTTP PUT is dead, long live HTTP PUT
published: true
description: Engineering client-server applications will always mean entertaining the possibility that clients will lag the server by at least one version.
---

![Hey, nice API. It'd be a shame if you ever wanted to update it.]({{ site.baseurl }}/public/hey-nice-api.jpg)

Absent some sort of alien quantum technology, engineering client-server applications will always mean entertaining the possibility that clients will lag the server by at least one version.

Most people I know prefer to use the oldest version of software they can get away with. Updating software without a specific feature in mind means spending precious time learning a new interface and dealing with new bugs.

As a system engineer, you can respond a couple different ways:

1. If you're ok with your users hating you, you can [force client updates](http://www.pushsquare.com/news/2019/12/ps4_firmware_update_7_02_is_ready_for_download_right_now).
2. Plan for the possibility that some clients will lag the server by at least one version.

### A problematic verb

When it comes to RESTful API's, the first HTTP verb that comes to mind when you're editing a entity is almost always PUT. The problem with using PUT is that it lacks forward compatibility. Here's why:

Step 1: Client v1.1 POSTs a new entity to the server:
``` json 
{
    "name": "Dave",
    "age": 47
}
```
Step 2: An older client (v1.0) GETs the entity client v1.1 created, transforms it to a model compatible with its UI, lets the user modify it, and then attempts to send it back to the server via HTTP PUT. The thing is, the 'age' property was just added in v1.1 and client v1.0 has no notion of it, so the payload it sends looks like this:
``` json
{
    "name": "Whitmer"
}
```

Because we're using PUT, the server must replace the entire document in the database with this new document, so "age" is dropped and this data is lost. If an older client is unknowingly deleting data created by newer clients, you're gonna have a bad time.

We can avoid having a bad time.

### Enter JSON Merge Patch

No, not [that other patch](https://tools.ietf.org/html/rfc6902). [JSON Merge Patch](https://tools.ietf.org/html/rfc7386) is perfect because it's almost exactly like PUT except it doesn't delete fields it doesn't know about.

In other words, when you omit a property in a Merge PATCH, the database will simply let that property remain whatever value it presently is. This provides forward compatibility and thus fixes the main issue we have with PUT.

### In closing, stop using HTTP PUT

It's just bad, mkay? Okay fine it's actually not that bad. PUT can actually be pretty great [for other things](https://stackoverflow.com/questions/630453/put-vs-post-in-rest), just consider PATCH instead when you're trying to edit entities.