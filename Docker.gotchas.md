**Docker has a hard limit of 42 AUFS layers**. Every time a command in
your Dockerfile gets executed (except the `FROM` command), the result is
a Docker image which is the equivalent to 1 AUFS layer. This sounds
great in theory as one can stack all those images to build complex
containers similar to how git trees get stacked within git commits.  The
speed boost from using images vs executing the same system commands over
and over again is immense, but there is one caveat. The `ADD` command
cannot be cached, so as soon as Docker comes across this command, it
stops using the pre-built images and will start creating new ones. This
is true for all commands that follow the `ADD` command. Furthermore,
this also continues in all inheriting Dockerfiles. In our [Ruby 2.0
Dockerfile](#xxx), where we were inheriting from [ruby-build](#xxx) ,
Ruby compilation would not be cached because `ADD` was used in the
ruby-build Dockerfile.


Many atomic commands (and thus images) results in a nice logical model
and is easier to understand when reading a Dockerfile. 

The flip-side of using many atomic commands (and thus images) is the 42
AUFS layers limit that I've mentioned. It might sound like a lot, but
I've hit it in the first implementation of this fairly simple
[dockerized Sinatra app](https://github.com/cambridge-healthcare/hi_sinatra-docker/tree/v0.1.0).

The error manifests itself through a rather vague error message:

<pre>
Error build: Unable to mount using aufs
Unable to mount using aufs
</pre>

This [docker github
issue](https://github.com/dotcloud/docker/issues/2028) goes into more
detail about it.

[Merging all
commands](https://github.com/cambridge-healthcare/dockerfiles/pull/1)
results in a single Docker image per Dockerfile. One clear advantage is
that now we can see exactly how much disk space each layer uses:

<pre>
REPOSITORY             TAG                 ID                  CREATED             SIZE
howareyou/ruby         2.0.0-p247          b712db79101d        3 hours ago         131.7 MB (virtual 613 MB)
howareyou/ruby-build   20130806            fb9df8396ee4        4 hours ago         203.4 MB (virtual 481.3 MB)
howareyou/ubuntu       12.04               a6bc0c7e68ea        4 hours ago         146.4 MB (virtual 277.9 MB)
ubuntu                 12.04               8dbd9e392a96        5 months ago        131.5 MB (virtual 131.5 MB)
</pre>

Docker ubuntu:12.04 image uses 131.5 MB. Our customized ubuntu:12.04
takes a further 146.4 MB and result in a combined total of 277.9 MB.  To
run ruby 2.0.0-p247 we need a further 2 Docker images: ruby-build and
the ruby image itself. A full ruby 2.0.0-p247 Docker image requires 4
images in total using 613 MB. Before combining commands across various
Dockerfiles, this same image used to be made of about 30+ Docker images,
bringing us very close to the 42 AUFS layers limit.

[Flat Docker images](http://3ofcoins.net/2013/09/22/flat-docker-images/)
by Maciej Pasternacki is a great post which talks about this same
problem more and offers a Perl script as a solution to the above
problem.
