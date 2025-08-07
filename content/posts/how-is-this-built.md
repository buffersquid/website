+++
title = "Website and Infrastructure"
description = "And why does it look like that?"
date = 2025-08-05
+++
## The Why
To understand how this website is built, you must first understand the desired deployment structure and outcome.

This site, and the computer that it's hosted on, are both built with [GNU Guix](https://guix.gnu.org/). If you've heard of [NixOS](nixos.org), it follows a very similar design (in fact, the Guix core uses the Nix core directly. If you look at the Guix source code, you'll see a `nix/` directory in the root!). Some of these include:
- Using Guile Scheme as its configuration language
    - Using Guile instead of the Nix language means that we can take full advantage of the configuration language being an actual, fully thought-out programming language first, which gives us access to all the Lisp-y stuff like macros.
- Being libre-by-default
    - I will admit that this can be a major headache depending on what kind of programs you are running, since by default, the main GNU channel can't have any non-free software.
- More rigorous bootstrapping process
    - For a package to be merged into the Guix ecosystem, the entire thing must be able to be bootstrapped up from a very small binary seed. This means that no packages in the main repo are binary blobs and can be verified as authentic.

I could go on and on about the neat things (and not-so-neat things) that Guix does, so maybe I'll do that in a future post. For now, back to the website.

## Website
As of the time of this post, the site is built with [Zola](https://www.getzola.org), which is a Rust-based static site generator.

As stated above, we want to use Guix to build this site, which means that we need to be able to pull in the source code for the website and build it with Zola, all in a pure, functional manner. But first, we need Zola to run on Guix. At the time of writing this post, Zola is **not** packaged in the main repo, and so we needed to package it ourselves. Luckily, Guix provides a nice way to transform Rust crates into Guix package definitions by using `guix import`.

```sh
guix import crate name@version -r
```

Running the above command will spit out a Guix package definition that you can feed into the daemon to produce a build. After doing this for many, many, *many* of Zola's dependencies, we were finally able to build the Zola package and add it to the [buffersquid channel](https://github.com/buffersquid/guix). A channel in Guix is like a repository for packages that aren't in Guix main yet. This could be because they are proprietary, or because they just aren't important enough to add.

Now that we have a successful build of Zola, we can move on to packaging this website using our new Zola package. The package definition looks like:

```scm
(define-module (buffersquid packages website)
  #:use-module (guix packages)
  #:use-module (guix git-download)
  #:use-module (guix build-system trivial)
  #:use-module ((guix licenses) #:prefix license:)
  #:use-module (buffersquid packages zola)
  #:export (website))

(define-public website
  (package
    (name "website")
    (version "0.0")
    (source (origin
          (method git-fetch)
          (file-name (git-file-name name version))
          (uri (git-reference (url "https://github.com/buffersquid/website.git")
                              (commit "794254ab8841b9c51a9f0b655f8696e8b72db046")))
          (sha256 (base32 "1l2i8hlmkbica8mqlr7xin51v3f1jk0rkdnqdny566mhcvl92c85"))))
    (native-inputs (list zola))
    (build-system trivial-build-system)
    (arguments
     `(#:modules ((guix build utils))
       #:builder (begin
                   (use-modules (guix build utils))
                   (let* ((output (assoc-ref %outputs "out"))
                          (source (assoc-ref %build-inputs "source"))
                          (zola   (string-append (assoc-ref %build-inputs "zola") "/bin/zola")))
                     (invoke zola "--root" source "build" "--output-dir" output "--minify")
                     #t))))
    (synopsis "Homepage for buffersquid")
    (description "Homepage for buffersquid")
    (home-page "https://buffersquid.com")
    (license license:gpl3+)))
```

Running this through
```sh
guix build
```
Gives us an output like:
```sh
guix build -L . website
    /gnu/store/0qwxmd2kj8l436i79z4xf9vfbn462fw3-website-0.0
ls /gnu/store/0qwxmd2kj8l436i79z4xf9vfbn462fw3-website-0.0
    index.html  robots.txt  style.css
```

Kinda neat! So we've now successfully taken our [website source code](https://github.com/buffersquid/website), fed it through the Guix daemon, and out pops a reproducible build!

Now that we have this build, we shift focus away from the code and onto the infrastructure.

## Infrastructure

The first step of getting a website running is getting a domain name so that your users don't need to remember the IP address of your host. I got mine off [Namecheap](https://www.namecheap.com/), but they all work pretty much the same way.

After getting your domain, you need to point it to the IP address where the site is hosted. This can get a little tricky. If you are paying for a VPS somewhere, then the IP address should be static, and you can simply point the domain name to that IP. However, if you are self-hosting **and** your ISP didn't give you a static IP (most don't), then you will need to set up DDNS (dynamic DNS) to update that pointer every time your ISP changes. There are a bunch of different ways to do this, but my router allows for DDNS, so the flow for me is like:

```
Domain name (buffersquid.com) -> DDNS domain (some long and illegible name) -> IP
```

You can verify that your system works by using the `dig` command:

```sh
dig +short your-domain-name.com
```
And you should see an IP address as the response.

After getting this working, you will need to open port 80 (HTTP port) and 443 (HTTPS port) on your router to allow incoming traffic to access these ports.

### Deploying via Guix

Now that we have both our website package and our domain name pointing to the correct IP, it's time to get our server ready to *serve* our website! To do this, we are, of course, going to be using Guix as the deploy tool, along with nginx, and certbot for the HTTPS stuff.

Here is the Guix code for accomplishing this:

```scm
(define-module (buffersquid system website)
  #:use-module (guix gexp)
  #:use-module (gnu services)
  #:use-module (gnu services shepherd)
  #:use-module (gnu services web)
  #:use-module (gnu services certbot)
  #:use-module (buffersquid packages website)
  #:export (website-base-services))

(define %website-name "buffersquid.com")
(define %website-dir (string-append "/srv/http/" %website-name))

(define website-deploy-service
  (simple-service
    'website-deploy
    shepherd-root-service-type
    (list
      (shepherd-service
        (requirement '(file-systems))
        (provision '(website-deploy))
        (documentation (string-append "Copy website out of store to @file{" %website-dir "}"))
        (start #~(let ((website-in-store #$website))
                   (lambda _
                     (mkdir-p #$%website-dir)
                     (copy-recursively website-in-store #$%website-dir))))
        (stop #~(lambda _
                  (with-exception-handler (lambda (e) (pk 'caught e))
                                          (lambda () (delete-file-recursively #$%website-dir))
                                          #:unwind? #t)
                  #f))))))

(define website-nginx-service
  (service nginx-service-type
           (nginx-configuration
             (server-blocks
               (list
                 (nginx-server-configuration
                   (listen '("443 ssl"))
                   (server-name (list %website-name))
                   (root %website-dir)
                   (ssl-certificate (string-append "/etc/letsencrypt/live/" %website-name "/fullchain.pem"))
                   (ssl-certificate-key (string-append "/etc/letsencrypt/live/" %website-name "/privkey.pem"))))))))

(define website-certification-service
  (service certbot-service-type
           (certbot-configuration
             (certificates
               (list
                 (certificate-configuration
                   (name %website-name)
                   (domains (list %website-name))))))))

(define website-base-services
  (list website-deploy-service
        website-nginx-service
        website-certification-service))
```

There are a couple of interesting things to note here.
1. You'll notice that we actually have 3 services being appended. Nginx, certbot, and a custom service called `website-deploy-service`. We need to have this `website-deploy-service` as certbot needs to be able to write to its root, and if we were to hand it a Guix package directly, it will fail because Guix derivations are immutable.

2. If you were to take this code, add it to your Guix system, then initialize a new machine with it, it will fail. This is because there seem to be some oddities in Guix when both certbot and nginx start at the same time, since they both actually modify the nginx service. So, when you initialize a new machine, just configure it with certbot first, then add the nginx configs. There may be a cool way of making this happen all at once, but I don't know of any way, and this workaround works well enough.

## Why does it look like that?
I'm not a designer. Expect the colors and style of this website to change massively in the future.
If you are coming to this site for its artistic merit... You should reconsider.
