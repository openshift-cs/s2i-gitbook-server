FROM centos:7

# Copy into place S2I builder scripts and label the Docker image so that
# the 's2i' program knows where to find them.

COPY s2i /usr/libexec/s2i

# Install any required system packages. We need the Apache httpd web
# server and various NodeJS packages in this instance, plus the 'rsync'
# package so we can copy files into the running container if necessary.

RUN rpmkeys --import file:///usr/libexec/s2i/RPM-GPG-KEY-CentOS-7 && \
    PACKAGES1="httpd rsync git" && \
    yum install -y --setopt=tsflags=nodocs ${PACKAGES1} && \
    rpm -V ${PACKAGES1} && \
    yum install -y centos-release-scl-rh && \
    PACKAGES2="rh-nodejs10 rh-nodejs10-npm rh-nodejs10-nodejs-nodemon" && \
    ln -s /usr/lib/node_modules/nodemon/bin/nodemon.js /usr/bin/nodemon && \
    yum install -y --setopt=tsflags=nodocs $PACKAGES2 && \
    rpm -V ${PACKAGES2} && \
    yum clean all -y && \
    rm -rf /var/cache/yum

# Create a non root account called 'default' to be the owner of all the
# files which the Apache httpd server will be hosting. This account
# needs to be in group 'root' (gid=0) as that is the group that the
# Apache httpd server would use if the container is later run with a
# unique user ID not present in the host account database, using the
# command 'docker run -u'.

RUN mkdir -p /opt/app-root && \
    chown 1001:0 /opt/app-root && \
    useradd -u 1001 -r -g 0 -d /opt/app-root -s /sbin/nologin \
            -c "Default Application User" default

# Modify the default Apache configuration to listen on a non privileged
# port of 8080, log everything to stdout/stderr, use different runtime
# directory and host files from under the account created to hold files.

RUN sed -i -e "s%^Listen 80$%Listen 8080%" \
           -e "s%logs/access_log%/proc/self/fd/1%" \
           -e "s%logs/error_log%/proc/self/fd/2%" \
           -e "s%/var/www/html%/opt/app-root/src/_book%" \
           /etc/httpd/conf/httpd.conf && \
    echo "DefaultRuntimeDir /opt/app-root" >> /etc/httpd/conf/httpd.conf && \
    echo "PidFile /opt/app-root/httpd.pid" >> /etc/httpd/conf/httpd.conf

EXPOSE 8080

# Install GitBook CLI to global SCL NodeJS installation.

RUN source scl_source enable rh-nodejs10 && \
    npm install -g gitbook-cli@2.3.2

LABEL io.k8s.description="S2I builder for hosting GitBook site." \
      io.k8s.display-name="GitBook Server" \
      io.openshift.expose-services="8080:http" \
      io.openshift.tags="builder,gitbook" \
      io.openshift.s2i.scripts-url="image:///usr/libexec/s2i"

RUN chown -R 1001:0 /usr/libexec/s2i && \
    find /usr/libexec/s2i -type d -exec chmod g+ws {} \; && \
    find /opt/app-root -type d -exec chmod g+ws {} \;

# Ensure container runs as non root account.

USER 1001

# Ensure HOME directory environment variable is set for when run as
# a uid with no entry in the host account database.

ENV HOME=/opt/app-root

# Install GitBook package into the home directory of the non root user.

RUN source scl_source enable rh-nodejs10 && \
    gitbook fetch latest

# Create the 'src' directory where the source code will reside. Fixup
# all the directories under the account so they are group writable to
# the 'root' group (gid=0) so they can be updated if necessary, such as
# would occur if using 'oc rsync' to copy files into a container.

RUN mkdir /opt/app-root/src && \
    find /opt/app-root -type d -exec chmod g+ws {} \;

# Ensure that build and execution of Apache httpd server is done from
# the 'src' directory where the source code will reside.

WORKDIR /opt/app-root/src

# Set the default command to be the 'usage' script as can't meaningfully
# run the container unless a 's2i' build has been done.

CMD [ "/usr/libexec/s2i/usage" ]
