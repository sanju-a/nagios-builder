
### ================================== ###
###   STAGE 1 CREATE BUILD IMAGE       ###
### ================================== ###

ARG ROCKY_LINUX_REL=8.8
ARG BUILDPLATFORM
ARG TARGETPLATFORM

FROM --platform=$BUILDPLATFORM rockylinux:${ROCKY_LINUX_REL} as zreis-builder-base

ENV TARGETPLATFORM=${TARGETPLATFORM:-linux/amd64} \
    BUILDPLATFORM=${BUILDPLATFORM:-linux/amd64}

RUN printf "Building for TARGETPLATFORM=${TARGETPLATFORM} on BUILDPLATFORM=${BUILDPLATFORM}\n" \
    && printf "With uname -s : " && uname -s \
    && printf "and  uname -m : " && uname -m 

### ============================================= ###
###   STAGE 2 COMPILE AND INSTALL NAGIOS SOURCES  ###
### ============================================= ###

FROM zreis-builder-base as zreis-nagios-compiler

LABEL maintainer="Sanju Abraham <sabraham@zscaler.com> and <asanju@gmail.com>"

ARG NAGIOS_VERSION=4.4.13
ARG NAGIOS_PLUGINS_VERSION=2.4.6
ARG MK_LIVESTATUS_VER=1.2.8p25

ARG NAGIOS_RPMS_PATH="/tmp/nagioscore-nagios-${NAGIOS_VERSION}/contrib/"
ARG MK_LIVESTATUS_OBJ_PATH="/tmp/mk-livestatus-${MK_LIVESTATUS_VER}/src/"

ENV NAGIOS_HOME=/opt/nagios \
    NAGIOS_USER=nagios \
    NAGIOS_GROUP=nagios \
    NAGIOS_CMDUSER=nagios \
    NAGIOS_CMDGROUP=nagios \
    NAGIOS_TIMEZONE=UTC \
    NAGIOS_FQDN=zreis.nagios.com \
    NAGIOSADMIN_USER=nagiosadmin \
    NAGIOSADMIN_PASS=nagios

# Add dependencies required to build Nagios
RUN dnf update -y && \
    dnf install -y automake libtool autoconf gnutls epel-release \
                   make gcc gcc-c++ autoconf gettext gnutls-devel \
                   gettext-devel kernel-headers-$(uname -r) openssl-devel net-snmp net-snmp-utils \
                   gd gd-devel perl postfix unzip wget httpd which procps-ng

RUN OS_VER=$( grep -E '^(VERSION)=' /etc/os-release ) && \
    if [[ $OS_VER == *"Green Obsidian"* ]]; then \
       dnf config-manager --set-enabled powertools; \
    elif [[ $OS_VER == *"Blue Onyx"* ]]; then \
       dnf config-manager --set-enabled crb; \
    else \
       echo "Unsupported OS Version $OS_VER"; \
    fi

# This step is safe to have for both rockylinu-8 and 9
RUN /usr/bin/crb enable

RUN dnf install -y perl-Net-SNMP perl-Test-Harness

# Download Nagios core, plugins and nrpe sources
RUN    cd /tmp && \
       echo -n "Downloading Nagios ${NAGIOS_VERSION} source code: " && \
       wget -O nagios-core.tar.gz "https://github.com/NagiosEnterprises/nagioscore/archive/nagios-${NAGIOS_VERSION}.tar.gz" && \
       echo -n -e "OK\nDownloading Nagios plugins ${NAGIOS_PLUGINS_VERSION} source code: " && \
       wget -O nagios-plugins.tar.gz "https://github.com/nagios-plugins/nagios-plugins/archive/release-${NAGIOS_PLUGINS_VERSION}.tar.gz" && \
       echo "Successfully downloaded the sources for nagios-${NAGIOS_VERSION} and nagios-plugins-${NAGIOS_PLUGINS_VERSION}"


#Setting group and user
RUN  OS_VER=$( grep -E '^(VERSION)=' /etc/os-release ) && \
     if [[ $OS_VER == *"Green Obsidian"* ]] || [[ $OS_VER == *"Blue Onyx"* ]]; then \
        groupadd ${NAGIOS_GROUP} && useradd ${NAGIOS_USER} -G ${NAGIOS_CMDGROUP} -g ${NAGIOS_USER}; \
     else \
        addgroup -S ${NAGIOS_GROUP} && adduser  -S ${NAGIOS_USER} -G ${NAGIOS_CMDGROUP} -g ${NAGIOS_USER}; \
     fi

# Compile Nagios Core
RUN    ls -l /tmp && cd /tmp && \
       tar zxf nagios-core.tar.gz && \
       tar zxf nagios-plugins.tar.gz && \
       cd  "/tmp/nagioscore-nagios-${NAGIOS_VERSION}" && \
       echo -e "\n ===========================\n  Configure Nagios Core\n ============================\n" && \
       ./configure \
            --prefix=${NAGIOS_HOME}                  \
            --exec-prefix=${NAGIOS_HOME}             \
            --enable-event-broker                    \
            --with-command-user=${NAGIOS_CMDUSER}    \
            --with-command-group=${NAGIOS_CMDGROUP}  \
            --with-nagios-user=${NAGIOS_USER}        \
            --with-nagios-group=${NAGIOS_GROUP}      && \
       echo -e "\n\n ===========================\n Configured Nagios Core\n =========================== \n\n" && \
       echo -e "\n\n ===========================\n Compile Nagios Core\n =============================\n\n" && \
       make all && \
       echo -e "\n\n ===========================\n Compiled Nagios Core\n ============================= \n\n" && \
       echo -e "\n\n ===========================\n  Install Nagios Core\n =============================\n" && \
       make install && \
       make install-commandmode && \
       make install-config && \
       make install-webconf && \
       echo -e "\n\n ===========================\n  Installed Nagios Core\n ============================ \n\n" && \
       echo -n "Nagios installed size: " && \
       du -h -s ${NAGIOS_HOME} && \
       echo -e "\n\n"

### ================================= ###
###   STAGE 3 CREATE RPM NAGIOS-CORE  ###
### ================================= ###
RUN dnf install -y rpmdevtools rpm-build policycoreutils-devel selinux-policy-devel perl-generators expect
RUN dnf --enablerepo=devel install -y gdbm-devel doxygen gperf perl
RUN echo -e "\n ===========================\n  Generating RPM Nagios Core\n ============================\n"
RUN cd "/tmp/nagioscore-nagios-${NAGIOS_VERSION}/contrib" && \
    yes | expect -c 'expect "\n" {spawn make rpm; expect -re "Reversed (or previously applied) patch detected!  Assume -R? \[n\]"; send -- "y\r"; expect -re "Apply anyway? \[n\]"; send -- "y\r"; set timeout -1; expect eof;}'
RUN echo -e "\n ===========================\n  Generated RPMs Nagios Core\n ============================\n"

### ============================================================= ###
###   STAGE 4 DOWNLOAD AND COMPILE MK-LIVESTATUS-1.2.8p25 SOURCES ###
### ============================================================= ###
# Download MK-LIVESTATUS-1.2.8p25
RUN    dnf install -y libstdc++-static
RUN    cd /tmp && \
       echo -n "Downloading source code for MK-LIVESTATUS-${MK_LIVESTATUS_VER}: " && \
       wget -O mk-livestatus.tar.gz "https://mirrors.xtom.com/freebsd-ports/distfiles/check_mk/mk-livestatus-${MK_LIVESTATUS_VER}.tar.gz" 

RUN    ls -l /tmp && cd /tmp && \
       tar zxf mk-livestatus.tar.gz && \
       cd  "/tmp/mk-livestatus-${MK_LIVESTATUS_VER}" && \
       sed -i '/syslog.h/a #include <pthread.h>' src/logger.h && \
       sed -i '/void logger/i extern pthread_t g_mainthread_id;\n' src/logger.h && \
       sed -i '/pthread_t g_mainthread_id;/d' src/module.c

RUN    cd /tmp/mk-livestatus-${MK_LIVESTATUS_VER} && \
       echo -e "\n ===========================\n  Configure MK-LIVESTATUS \n ============================\n" && \
       ./configure \
            CPPFLAGS=-I/tmp/nagioscore-nagios-${NAGIOS_VERSION}/include/ --with-nagios4 && \ 
       echo -e "\n\n ===========================\n Configured MK-LIVESTATUS \n =========================== \n\n" && \
       echo -e "\n\n ===========================\n Compile MK-LIVESTATUS\n =============================\n\n" && \
       make && \
       echo -e "\n\n ===========================\n Compiled MK-LIVESTATUS\n ============================= \n\n"


### ==================================================== ###
###   STAGE 5 COMPILE AND INSTALL NAGIOS-PLUGINS SOURCES ###
### ==================================================== ###

# Compile Nagios Plugins
RUN    echo -e "\n\n ===========================\n  Configure Nagios Plugins\n ===========================\n" && \
       cd  /tmp/nagios-plugins-release-${NAGIOS_PLUGINS_VERSION} && \
       ./autogen.sh && \
       ./configure  --with-nagios-user=${NAGIOS_USER} \
                    --with-nagios-group=${NAGIOS_USER} \
                    --with-openssl \
                    --with-ps-command="/bin/ps axwo 'stat uid pid ppid vsz rss pcpu etime comm args'" \
                    --prefix=${NAGIOS_HOME}        && \
       echo -e "\n\n ===========================\n  Configured Nagios Plugins\n =========================== \n\n" && \
       echo -n "Replacing \"<sys\/poll.h>\" with \"<poll.h>\": " && \
       egrep -rl "\<sys\/poll.h\>" . | xargs sed -i 's/<sys\/poll.h>/<poll.h>/g' && \
       egrep -rl "\"sys\/poll.h\"" . | xargs sed -i 's/"sys\/poll.h"/"poll.h"/g' && \
       echo "Replacing: OK" && \
       echo -e "\n\n ===========================\n Compile Nagios Plugins\n ===========================\n" && \
       make && \
       echo -e "\n\n ===========================\n Compiled Nagios Plugins\n =========================== \n\n" && \
       echo -e "\n\n ===========================\nInstall Nagios Plugins\n =============================\n" && \
       make install && \
       echo -e "\n\n ===========================\nInstalled Nagios Plugins\n =============================\n" && \
       echo "Nagios plugins installed successfully: OK"


# Compile Nagios files
# Create SSMTP configuration

RUN sed -i 's,/bin/mail,/usr/bin/mail,' ${NAGIOS_HOME}/etc/objects/commands.cfg  && \
    sed -i 's,/usr/usr,/usr,'           ${NAGIOS_HOME}/etc/objects/commands.cfg  && \
                                                                                    \
    : '# Modify Nagios mail commands in order to work with SSMTP'         && \
    sed -i 's/^.*command_line.*Host Alert.*$//g' /opt/nagios/etc/objects/commands.cfg && \
    sed -i 's/^.*command_line.*Service Alert.*$//g' /opt/nagios/etc/objects/commands.cfg && \
    sed -i '/notify-host-by-email/a command_line /usr/bin/printf "%b" "Subject: $NOTIFICATIONTYPE$ Host Alert: $HOSTNAME$ is $HOSTSTATE$\\n\\n***** Nagios *****\\n\\nNotification Type: $NOTIFICATIONTYPE$\\nHost: $HOSTNAME$\\nState: $HOSTSTATE$\\nAddress: $HOSTADDRESS$\\nInfo: $HOSTOUTPUT$\\n\\nDate/Time: $LONGDATETIME$\\n" | /usr/sbin/sendmail -v $CONTACTEMAIL$' ${NAGIOS_HOME}/etc/objects/commands.cfg  && \
    sed -i '/notify-service-by-email/a command_line /usr/bin/printf "%b" "Subject: $NOTIFICATIONTYPE$ Service Alert: $HOSTALIAS$/$SERVICEDESC$ is $SERVICESTATE$\\n\\n***** Nagios *****\\n\\nNotification Type: $NOTIFICATIONTYPE$\\n\\nService: $SERVICEDESC$\\nHost: $HOSTALIAS$\\nAddress: $HOSTADDRESS$\\nState: $SERVICESTATE$\\n\\nDate/Time: $LONGDATETIME$\\n\\nAdditional Info:\\n\\n$SERVICEOUTPUT$\\n" | /usr/sbin/sendmail -v $CONTACTEMAIL$' ${NAGIOS_HOME}/etc/objects/commands.cfg

RUN echo "use_timezone=${NAGIOS_TIMEZONE}" >> ${NAGIOS_HOME}/etc/nagios.cfg && \
    sed -i 's/date_format=us/date_format=iso8601/g' ${NAGIOS_HOME}/etc/nagios.cfg

RUN dnf clean all && \
    rm -rf /var/cache/dnf

### ==================================================== ###
###   STAGE 6 COPY RPMS and MK_LIVESTATUS OBJECTS        ###
### ==================================================== ###
 
FROM zreis-nagios-compiler
WORKDIR ${NAGIOS_RPMS_PATH}
COPY *.rpm .
WORKDIR $MK_LIVESTATUS_OBJ_PATH
COPY livestatus.so .
COPY livestatus.o .
COPY unixcat .
COPY unixcat.o .