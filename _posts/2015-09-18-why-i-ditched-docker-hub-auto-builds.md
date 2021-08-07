---
layout: post
title:  Why I Ditched DockerHub's Automated Builds
date:   2015-09-18 12:00:00
categories: Docker
preview: If you've created Docker containers and put them on the central registry then you know that you can have DockerHub do the builds for you using the "Automated Build" feature. This is a great feature for maintaining relatively stable projects with infrequent changes but if you're living the agile life and iterating quick and often, you're going to find yourself wanting more.
og_type: article
og_description: If you've created Docker containers and put them on the central registry ([http://hub.docker.com](http://hub.docker.com)) then you know that you can have DockerHub do the builds for you using the "Automated Build" feature. This is a great feature for maintaining relatively stable projects with infrequent changes but if you're living the agile life and iterating quick and often, you're going to find yourself wanting more.
disqus_id: 13
---

If you've created Docker containers and put them on the central registry then you know that you can have DockerHub do the builds for you using the "Automated Build" feature. This is a great feature for maintaining relatively stable projects with infrequent changes but if you're living the agile life and iterating quick and often, you're going to find yourself wanting more.

What do I mean by "wanting more"? Well, first of all, the automated builds are great if you want to keep all of your iterations on the `latest` tag. Which for most, is exactly what you will want. You then manually push named tags (such as `0.1` or `1.0`) or, alternatively tag the release on GitHub and go and update the build settings to build that new tag and map that Git tag to your Docker tag. For most projects this is exactly what you want to do. There is still the manual step of logging into DockerHub and updating the build settings with the new tag which is kind of annoying but not a show stopper.

<img class="img-fluid" src="/img/postimgs/automated_build.png"/>

What this should really be (Docker, pay attention), is automatic!! Yea, it should build all Git tags automatically unless overridden. Or, even better, a regex would be awesome. The build settings should let me say, "build every tag that matches `release-(.*)` as `$1`" ... wouldn't that be great? Yea, it doesn't do that. (hint hint) but I digress...

In my case, I need an auto versioning of my containers so that I create a new version on DockerHub for each merge to master for my particular project. This is why I ditched the automated build stuff altogether. The idea is to have a base version like `1.0.0` and then every merge to master will build a point version on top of it like `1.0.0.0` and `1.0.0.1` etc. This is more appropriate for agile style end-user apps (like webapps and such) because I have many build versions in the registry that I can choose from if I need to rollback or something similar. So for this case, it requires building using my own CI system. For this, I chose [CircleCI](http://circleci.com) because of their awesome Docker support.

So, first things first, we need to setup a way of versioning our container and get the next version available. I put a `VERSION` file at the top-level of my repo and put the base version there. Then I wrote some ruby code to

  * read the VERSION file
  * hit the DockerHub api and get the current tags
  * then figure out the next available tag for the current base version

~~~ ruby
require 'httparty'

# Read the base version from VERSION file.
def version
  File.readlines('./VERSION').first.strip
end

# The name of the container
# This is based off of the directory name
def container_name
  File.basename(Dir.getwd)
end

# The username the container is pushed to on DockerHub
def username
  'parabuzzle'
end

# Get the latest version for the given base version provided by #version
def hub_version
  base           = version
  taginfo        = JSON.parse(HTTParty.get("https://hub.docker.com/v2/repositories/#{username}/#{container_name}/tags/").body)['results']
  return {base: base, build: nil} if taginfo.nil?
  tags = []
  taginfo.each do |tag|
    tags << tag['name']
  end
  current_base   = tags.grep(/#{base}/)
  return {base: base, build: nil} if current_base.empty?
  build = current_base.sort { |x,y|
      a = x.split('.')[base.split('.').count].to_i
      b = y.split('.')[base.split('.').count].to_i
      a <=> b
    }.last.split('.').last.to_i
  return {base: base, build: build}
end

# return current hub version for the current base
def latest_hub_version
  latest = hub_version
  "#{latest[:base]}.#{latest[:build]}"
end

# return the next version for the current base
def next_version
  latest = hub_version
  base   = version
  build  = latest[:build] || -1
  build  += 1
  "#{base}.#{build}"
end
~~~

As you can see here, this just uses the DockerHub API to handle versioning of my container. I put all this into a Rakefile with the following tasks and :boom: we're ready for automation!

~~~ ruby

task :install_deps do
  sh 'gem install bundler'
  sh 'bundle install'
end

desc "tags latest as next_version"
task :tag do
  sh "docker tag -f #{username}/#{container_name}:latest #{username}/#{container_name}:#{next_version}"
end

desc "pushes the next_version and latest to docker hub"
task :push => :tag do
  sh "docker push #{username}/#{container_name}:#{next_version}"
  sh "docker push #{username}/#{container_name}:latest"
end

desc "builds as latest"
task :build => :install_deps do
  sh "docker build -t #{username}/#{container_name}:latest ."
end

task :default => [:build, :push]
~~~

Now we're ready to just run `rake build` and `rake push`. I just wired that into the circle.yml for CircleCI like so:

~~~
machine:
  services:
    - docker

dependencies:
  override:
    - docker info
    - gem install httparty
    - rake build

test:
  override:
    - docker run -d -p 80:80 parabuzzle/docker-circle-ci:latest; sleep 10
    - curl --retry 10 --retry-delay 5 -v http://localhost:80/

deployment:
  hub:
    branch: master
    commands:
      - docker login -e $DOCKER_EMAIL -u $DOCKER_USER -p $DOCKER_PASS
      - rake push
~~~

And I'm done. Now I get auto versioning of my containers with merges to master.

In conclusion, automated builds on DockerHub are awesome but just not powerful enough for all use cases. By moving to your own CI system, you can control how and when your containers are built and distributed to the registry. In my case, I built a simple auto versioning system but you may want to make yours more intelligent by basing off of tags, or reading commit messages or a Changlog file. The options are endless and the coding is fun! So go knock yourself out.

I made a [template on GitHub](http://github.com/parabuzzle/docker-circleci-template) if you want to just copy that and replace my username with your username in the files :smiley:
