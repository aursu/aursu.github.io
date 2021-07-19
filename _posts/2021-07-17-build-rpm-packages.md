---
layout: post
title:  "Build RPM packages"
date:   2021-07-17 14:25:00 +0200
categories: rpm build
---

For a long time I use CentOS to run WEB application or/and Hosting services

There is `CentOS 7` which is LTS version till mid 2024 according to [product specifications](https://wiki.centos.org/About/Product). Which is quite enough time in order to decide where to go - either upgrade to  `CentOS Stream` or use other OS solutions (eg [OpenSUSE](https://get.opensuse.org/leap/) or any of replacement projects like [AlmaLinux](https://almalinux.org), [Rocky Linux](https://rockylinux.org), [Oracle Linux](https://www.oracle.com/linux/) etc).

At least in 2023-2024 it would be more obvious if CentOS 8 replacement distributions are successfull, well supported with strong community or not.

I don't see any troubles to use `CentOS Stream`. Of course considering their new model of distribution it would be more important to shift attention to OS patch management process within organisation. But in general it is very positive movement as it will introduce faster access to never and better versions of software out of the box. Which in turn will eliminate the need to build own/custom packages, support custom corporate repositories, OS builds and customizations etc.

Outdated stock packages are always a trouble. For example, in CentOS 7 default version of PHP is 5.4.16.
Nowadays it is hard even to find PHP-based applications which support this version of PHP. The same btw valid for rsyslog, systemd, kernel etc. In case of PHP there is a great solution like [Remi's RPM repository](https://rpms.remirepo.net) but in case of `kernel` or `systemd` it is not so easy to find never versions.

Therefore the best way is to upgrade to `CentOS 8` of course, but otherwise the only solution is to use either upstream versions of software, which, for example, works quite well in case of [Rsyslog](https://www.rsyslog.com/rhelcentos-rpms/), [Nginx](http://nginx.org/en/linux_packages.html#RHEL-CentOS), [OpenRresty](https://openresty.org/en/linux-packages.html#centos), or to build own packages using [RPM build process](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html-single/rpm_packaging_guide/index).

Another case to use RPM build is own software distribution.

The trouble with rpmbuild process has always been the build environment isolation within build server.

For example, usually it is bad idea to build `httpd` package on the server where recently has been done `php` build (without pre-build server cleanup). Any PHP build usually requires a lot of dependencies in order to provide multiple useful and just-in-case extensions to its users. Apache as a web/application service could require PHP itself but not whole set of its dependencies and in general it is very minimalystic. But there are autoconfiguration scripts within both software distributions (made with [autotools](https://www.gnu.org/software/automake/manual/html_node/Autotools-Introduction.html)) which try to detect provided by build environment features automatically as much as possible and include them into final build. As a result Apache will include some unexpected modules (which have been found by autoconf). In turn RPM package manager will require additional dependencies in order to install such `httpd` package - which also unexpected. It is usually not a trouble if PHP and Apache are installed on the same host and use same RPM repositories. But separate Apache installation could produce unexpected RPM dependencies issues especially if PHP build uses separate repositories set.

It was also very annoying to use separate build environment for RPM build. Especially when some patches are under development or separate build configuration should be created nad tested. Fedora project has separate framework for RPM packaging organisation - [Koji](https://fedoraproject.org/wiki/Koji) which uses [Mock](https://github.com/rpm-software-management/mock/wiki) for chroot isolation of build process. But I think it is suitable when you build a lot of packages for multiple architectures but not few software pieces on top of standard OS distribution.

All described issue has been resolved using `Docker` and `Docker Compose`. Build environment isolation and even better - minimalistyc build system which could be reproduced easily on any environment (Linux PC, Mac laptop, Windows workstation etc). This is the main benefit - so no way leftovers after previous build could affect current build process. Also using `Docker Compose` it is posssible to configure build workflow for parallel build of multiple configurations for multiple OSes.

This approach also provided ability to use CI/CD platforms such as `Circle CI`, `Travis CI`, `Github Actions` and finally `Jenkins CI` and `GitLab CI` in order to automate RPM build and deployment into repositories. Later this process has been extended with distribution of Docker images with preinstalled software for Web applications development and production operation.

...







