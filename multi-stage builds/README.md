Mastering Multi-Stage Builds
----------------------------

# 1. Table of Contents
[1. Table of Contents](#user-content-1-table-of-contents)\
[2. Intentions](#user-content-2-intentions)\
&nbsp; &nbsp; [2.1. Purging Secrets](#user-content-21-purging-secrets)\
&nbsp; &nbsp; [2.2. Unsequencing Layers](#user-content-22-unsequencing-layers)\
[3. Using Images as Layer Cache](#user-content-3-using-images-as-layer-cache)\
[4. Everything is About Efficiency (Caching)](#user-content-4-everything-is-about-efficiency-caching)\
&nbsp; &nbsp; [4.1. First Guess](#user-content-41-first-guess)\
&nbsp; &nbsp; [4.2. Wrongly Matched Layers](#user-content-42-wrongly-matched-layers)\
&nbsp; &nbsp; [4.3. Be Accurate with ```from-cache```](#user-content-43-be-accurate-with-from-cache)\
[5. Conclusion](#user-content-5-conclusion)

# 2. Intentions

With multi-stage builds following intentions can be achieved ...

## 2.1. Purging Secrets

If a secret is needed during the container build, this secret is usually part of each layer. To purge out this secret, you might use a dedicated layer to get rid of this secret afterwards.

Let's think about such a Dockerfile:
````
FROM debian
ENV cred=...
RUN wget https://$cred:servername/path/download.zip && do_some_stuff_with_download_zip
RUN do_some_more_stuff
````

Rephrasing the Dockerfile with multi-stage builds could look like this:
````
FROM debian AS builder
ENV cred=...
RUN wget https://$cred:servername/path/download.zip && do_some_stuff_with_download_zip

FROM debian
COPY --from=builder /any_folder /any_folder
RUN do_some_more_stuff
````

With that, all layers inside the builder stage will be consolidated to one layer and credentials given will have been purged out of the layers!


## 2.2. Unsequencing Layers
Layers are always ordered. Changing a layer causes that the changed layer and all subsequent layers are rebuilt. However, subsequent layers aren't necessarly dependent from previous ones. In such situations the subsequent layers are rebuild unneeded, wasting time.

Such a Dockerfile might look like the following:
````
FROM debian
RUN lot_of_commands_1
RUN lot_of_commands_2_indepedent_of_1
RUN lot_of_commands_3_indepedent_of_1_and_2
````

The above Dockerfile can be rephrased as the following:
````
FROM debian AS stage1
RUN lot_of_commands_1

FROM debian AS stage2
RUN lot_of_commands_2_indepedent_of_1

FROM debian AS stage3
RUN lot_of_commands_3_indepedent_of_1_and_2

FROM debian
COPY --from=stage1 /folder1 /folder1
COPY --from=stage2 /folder2 /folder2
COPY --from=stage2 /folder3 /folder3
````

Sure, the final stage will be always rebuilt at changes, but all intermediate stages will only be rebuilt as needed. By this, the build process is using more cached layers as before and skipping the rebuild of layers.

# 3. Using Images as Layer Cache
On build servers, it might be annoying that the build environment is always clean and so, no layers from former builds are existing.
For really complex builds, this needs to migitated, which can be done by storing the created staged layers into dedicated images.

Following Dockerfile is given:
````
FROM debian AS stage1
RUN lot_of_commands_1

FROM debian AS stage2
RUN lot_of_commands_2_indepedent_of_1

FROM debian
COPY --from=stage1 /folder1 /folder1
COPY --from=stage2 /folder2 /folder2
````

To store that stages into dedicated images, you might execute following commands:
```
docker build --target stage1 -t project:stage1
docker build --target stage2 -t project:stage2
docker build --from-cache project:stage1 --from-cache project:stage2 -t project:final
```

The parameter ```target``` means that *all* stages *until* this target stage will be executed.

The parameter ```from-cache```means that a dedicated image is used for acting as persistent layer cache.

In fact, the command block above is not complete. When you start using the parameter ```from-cache``` the local layer cache is not used anymore and all used images have to be listed. If you miss one image in the list, the affected stage will always be rebuilt.
Additionally ```docker pull/push``` commands are missing to make this example complete.

So the correct commands are:
```
docker pull project:stage1 || true
docker pull project:stage2 || true
docker pull project:final || true

docker build --target stage1 --from-cache debian:latest --from-cache project:stage1 -t project:stage1
docker build --target stage2 --from-cache debian:latest --from-cache project:stage2 -t project:stage2
docker build --from-cache debian:latest --from-cache project:stage1 --from-cache project:stage2 --from-cache project:final -t project:final

docker push project:stage1
docker push project:stage2
docker push project:final
```

>To simplify the later examples, the pull and push commands of the images are ommitted.


# 4. Everything is About Efficiency (Caching)

Unfurtunately there are situations where the layer caching is not working (as expected) either you are not accurate enough (e.g. in listing the involved images in the ```from-cache``` parameters) or Docker is not recognizing the correct cached layer and enforces a rebuild.

In this chapter, I will describe some pitfalls.

## 4.1. First Guess

Dockerfile:
```
FROM debian AS stage1
ENV DEBIAN_FRONTEND=noninteractive
RUN touch /tmp/stage1.txt && echo -e "\e[91m1: not cached\e[0m"

FROM tmp/multistage/01:stage_stage1 AS stage2
ENV DEBIAN_FRONTEND=noninteractive
RUN touch /tmp/stage2.txt && echo -e "\e[91m2: not cached\e[0m"

FROM debian
ENV DEBIAN_FRONTEND=noninteractive
COPY --from=stage1 /tmp/* /tmp/
COPY --from=stage2 /tmp/* /tmp/
RUN ls -l /tmp/ && echo -e "\e[91m3: not cached\e[0m"
```

docker-compose.yml:
```
version: '3.6'
services:

  stage1:
    build:
        context: .
        cache_from:
        - debian:latest
        - tmp/multistage/01:stage_stage1
        target: stage1        
    image: tmp/multistage/01:stage_stage1
    
  stage2:
    build:
        context: .
        cache_from:
        - debian:latest
        - tmp/multistage/01:stage2
        target: stage2
    image: tmp/multistage/01:stage_stage2

  final:
    build:
        context: .
        cache_from:
        - debian:latest
        - tmp/multistage/01:stage_stage1
        - tmp/multistage/01:stage_stage2
        - tmp/multistage/01:final
    image: tmp/multistage/01:final

    
# docker rmi tmp/multistage/01:stage_stage1 tmp/multistage/01:stage_stage2 tmp/multistage/01:final && docker system prune
```

A ```docker-compose build``` will run into  following situation:

|Build|Target Stage|Stage 1|Stage 2|Final Stage|
|-----|------------|-------|-------|-----------|
| initial | stage1 | built | - | - |
| initial | stage2 | :heavy_exclamation_mark:built | built | - |
| initial | final | cached | cached | built |
| subsequent | stage1 | cached | - | - |
| subsequent | stage2 | cached | cached | - |
| subsequent | stage2 | cached | cached | :heavy_exclamation_mark:built |

> All red marked builds are not intended!

> Cleanup with ```docker rmi tmp/multistage/01:stage_stage1 tmp/multistage/01:stage_stage2 tmp/multistage/01:final && docker system prune```

So, for some reasons layer caching is not working as expected, but continue reading the next chapter ...

## 4.2. Wrongly Matched Layers

Somehow Docker uses the first line after the ```FROM``` line to match a potential cached layer. If two layers have the same line after ```FROM``` a potential cached layer is wrongly assigned to the currently inspected layer and the current layer is getting rebuilt.

To ensure that the first line after ```FROM``` is unique, here an adjusted ```Dockerfile```:
```
FROM debian AS stage1
ENV DEBIAN_FRONTEND=noninteractive LAYER=stage1
RUN touch /tmp/stage1.txt && echo -e "\e[91m1: not cached\e[0m"

FROM tmp/multistage/01:stage_stage1 AS stage2
ENV DEBIAN_FRONTEND=noninteractive LAYER=stage2
RUN touch /tmp/stage2.txt && echo -e "\e[91m2: not cached\e[0m"

FROM debian
ENV DEBIAN_FRONTEND=noninteractive LAYER=final
COPY --from=stage1 /tmp/* /tmp/
COPY --from=stage2 /tmp/* /tmp/
RUN ls -l /tmp/ && echo -e "\e[91m3: not cached\e[0m"
```

A ```docker-compose build``` will run into  following situation:

|Build|Target Stage|Stage 1|Stage 2|Final Stage|
|-----|------------|-------|-------|-----------|
| initial | stage1 | built | - | - |
| initial | stage2 | :heavy_exclamation_mark:built | built | - |
| initial | final | cached | cached | built |
| subsequent | stage1 | cached | - | - |
| subsequent | stage2 | cached | cached | - |
| subsequent | stage2 | cached | cached | cached |

> Cleanup with ```docker rmi tmp/multistage/02:stage_stage1 tmp/multistage/02:stage_stage2 tmp/multistage/02:final && docker system prune```

Now, the caching behavior is improved, but keep reading ...


## 4.3. Be Accurate with ```from-cache```

Ok, I warned you and we weren't accurate with the ```from-cache``` parameters, so next try with an adjusted ```docker-compose.yml``` which contains all images/stages for stage2:

```
version: '3.6'
services:

  stage1:
    build:
        context: .
        cache_from:
        - debian:latest
        - tmp/multistage/03:stage_stage1
        target: stage1        
    image: tmp/multistage/03:stage_stage1
    
  stage2:
    build:
        context: .
        cache_from:
        - tmp/multistage/03:stage_stage1
        - tmp/multistage/03:stage_stage2
        target: stage2
    image: tmp/multistage/03:stage_stage2

  final:
    build:
        context: .
        cache_from:
        - debian:latest
        - tmp/multistage/03:stage_stage1
        - tmp/multistage/03:stage_stage2
        - tmp/multistage/03:final
    image: tmp/multistage/03:final
```

A ```docker-compose build``` will run into  following situation:

|Build|Target Stage|Stage 1|Stage 2|Final Stage|
|-----|------------|-------|-------|-----------|
| initial | stage1 | built | - | - |
| initial | stage2 | cached | built | - |
| initial | final | cached | cached | built |
| subsequent | stage1 | cached | - | - |
| subsequent | stage2 | cached | cached | - |
| subsequent | stage2 | cached | cached | cached |


> Cleanup with ```docker rmi tmp/multistage/03:stage_stage1 tmp/multistage/03:stage_stage2 tmp/multistage/03:final && docker system prune```


Finally we mastered the layer caching!

# 5. Conclusion

Docker layout caching is very powerful and can be fine tuned with multi-stage builds and extra parameters like ```target``` and ```cache-from```. However some tricky things like the first line after ```FROM``` have to be considered.
Some people in Docker forums state that also the order of images listed in ```cache-from``` parameters also affects the caching behavior but in my tests this order did not affect the outcome.
\
\
\
*Happy containerization!*

