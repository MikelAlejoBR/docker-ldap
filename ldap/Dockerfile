# Install the packages and clean apt lists
FROM debian:stretch as base

RUN apt-get update && \
   # Set non interactive to avoid slapd configuration prompts
   DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
   # For LDAP
   slapd=2.4.44+dfsg-5+deb9u2 \
   ldap-utils=2.4.44+dfsg-5+deb9u2 \
   ldapscripts=2.0.7-2 \
   # For envsubst
   gettext-base=0.19.8.1-2 \
   && apt-get clean \
   && rm -rf /var/lib/apt/lists/*

#########
# Build #
#########
# - Creates a default database
# - Creates a configuration for LDAP
# - Imports the data to the database
# - Removes the default created admin user
##
FROM base as build

COPY config/ /tmp/config
COPY data/ /tmp/data

# Set the build environment variables for envsubst
ARG DOMAIN
ARG ORGANIZATION
ARG ROOT_PASSWORD

# Reconfigure the slapd to create a new database to which the data will be
# imported
RUN envsubst < /tmp/config/debconf.conf.template > /tmp/config/debconf.conf \
   && debconf-set-selections /tmp/config/debconf.conf \
   && dpkg-reconfigure -f noninteractive slapd

# Set the build environment variables for envsubst
ARG ROOT_USER
ARG ROOT_PASSWORD_HASHED
ARG SUFFIX

# Remove the default schema and configuration
RUN rm -rf /etc/ldap/slapd.d/* \
   && envsubst < /tmp/config/slapd.conf.template > /tmp/config/slapd.conf \
   # Generate the LDAP configuration from the file
   && slaptest -v -f /tmp/config/slapd.conf -F /etc/ldap/slapd.d/

# Run slapd to be able to perform the operations
RUN slapd \
   # Import the data
   && ldapadd -xc -D "${ROOT_USER},${SUFFIX}" -w "${ROOT_PASSWORD}" -f /tmp/data/data.ldif \
   # Remove the default admin user
   && ldapdelete -D "${ROOT_USER},${SUFFIX}" -w "${ROOT_PASSWORD}" "cn=admin,${SUFFIX}"

# Remove all build related files
RUN rm -rf /tmp/config /tmp/data \
   # envsubst is no longer needed
   apt-get purge gettext-base

# Leave the entrypoint with the debug level 256
CMD ["/usr/sbin/slapd", "-d", "256"]
