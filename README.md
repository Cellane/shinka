# Shinka 神火

[<img src="https://asciinema.org/a/328564.svg" align="right" width="33%">](https://asciinema.org/a/328564)

> Tool for mass-updating Dokku applications deployed as Docker images from Docker Hub or private repositories (using `dokku tags:deploy`), and for cleaning old and unused images.

## Requirements

Shinka needs to be executed on the same host as your Dokku installation. Beyond access to Dokku’s binaries, Shinka requires the following software to operate properly:

- Docker
- Dokku
- Bash 4.3
- `awk`, `base64`, `curl`, `jq`, `whiptail` and a few other essential utilities find in most systems

Please install all the required tools before attempting to run Shinka.

## Configuration

Shinka will need to be configured before it can find updates for your applications deployed to Dokku. By default, when executing Shinka without any command-line arguments, it will expect to find the image configuration in a file names `images`.

You can, optionally, supply another configuration file name. For example, if you launch Shinka as `shinka unlocked`, configuration from file `unlocked.images` will be read instead.

Typical use case for switching configuration files is having the apps in the default configuration file locked to a tested major version (or a combination of major & minor version, if you want to play it extra-safe), and then have additional configuration file that you test once in a while that has much more lax locking rules, enabling you to discover newly released major versions of the apps that you’re running on your Dokku installation.

The configuration file unfortunately uses Bash syntax, and I’m terribly sorry for that. Trust me, I suffered more than you will.

You’re expected to define three associative arrays in the configuration file: `images`, `basicAuthImages` and `jwtTokenImages`. `images` contains information about apps that are updated from Docker Hub and don’t require any sort of authentication. `basicAuthImages` is used for images that require you to supply credentials via HTTP Basic Auth – for example, Google Container Registry. Finally, `jwtTokenImages` holds configuration about images that are pulled from registries that first require you to hit an endpoint that exchanges credentials for JWT token, then lets you access version information.

While the expected values differ in each of the three arrays, the key is always the same: it should equal to an application name in Dokku. The individual values in the value string are separated by semicolon. Let’s have a more detailed look at the examples shown in the `images.example` configuration file.

Starting from the top, the images found on Docker Hub are defined like this:

```bash
images=(
  ["blog"]='https://registry.hub.docker.com/v2/repositories/library/ghost/tags?page_size=1024&name=3;^3\(\.[[:digit:]]\+\)\+-alpine$'
  ["matomo"]='https://registry.hub.docker.com/v2/repositories/library/matomo/tags?page_size=1024;^[[:digit:]]\+\(\.[[:digit:]]\+\)\+$'
)
```

As you can see, two applications are defined – `blog` using Ghost’s official image, and `matomo` using Matomo’s official image. The first value in the value string is API endpoint that shows (hopefully) all image tags defined for wanted image. The second value in the value string (after the semicolon) is a tag regex. Only image tags that match the given regex are considered for updating. This allows you, for example, to only download updates that are based on the Alpine Linux (as you can see in the case of `blog` app).

In the example above, the regex is also used to keep Ghost version on the 3.x major version.

Because this is Bash and Bash is Bash, please make sure to escape any character that could be confusing to Bash properly. A lot of characters are confusing for Bash.

(A word of note about Docker Hub’s URLs: if an image is an official and verified image, the URL will begin with `/v2/repositories/library/<image-name>`. In the case of any other image, the URL will begin with `/v2/repositories/<username>/<image-name>`)

## Use flow

You can see a quick demo of a slightly older version of Shinka on [asciinema](https://asciinema.org/a/328564).

In general, these are the steps that you can expect Shinka to perform for you:

1. Authenticate as root user, if needed
2. Check Docker image tag data for all images defined in the configuration file
3. Present a summary of found updates, allowing you to individually de-select unwanted updates
4. Deploy selected updates
5. Offer to run a clean-up procedure, if updates were deployed
6. Wait for old Dokku containers to terminate, if needed
7. Find unused images
8. Present a summary of found unused images, allowing you to individually de-select images you want to keep
9. Remove selected images

## A few warnings

- I use Shinka on my personal server and while it works well for me, I can’t guarantee the same for everyone.
- Please pay attention when Shinka shows you the list of updates it wants to deploy. It doesn’t show the screen because it wants to look cool, it shows it because version comparing can be tricky and it wants your confirmation. Also, upstream version tags might be occasionally incorrect – I’m looking at you, cAdvisor.
- When removing images, Shinka will never try to remove images used by any running container. Still, please be careful with the cleanup functionality, as any image that will appear unused (and is not explicitly ignored at the top of the `shinka-cleanup` script) will be removed.
- Shinka, just like its author, is very strongly opinionated. Shinka will alter Docker image tags to be more aesthetically pleasing. For example, Shinka will turn image tag `3.15.3-alpine` into `3.15.3`, or tag `1.19.3.2764-ef515a800-ls94` into `1.19.3.2764`.
