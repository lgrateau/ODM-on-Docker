ARG FROMLIBERTY
ARG FROMLIBERTYBUILD
FROM maven:3.5.2-jdk-8-alpine AS builder
ARG ODMDOCKERDIR
ENV ODMDOCKERDIR $ODMDOCKERDIR
ENV SCRIPT /script
ENV APPS /config/apps
COPY $ODMDOCKERDIR/welcomepage /welcomepage
COPY $ODMDOCKERDIR/common/script $SCRIPT
COPY $ODMDOCKERDIR/common/drivers /config/resources
COPY $ODMDOCKERDIR/common/features $SCRIPT
COPY $ODMDOCKERDIR/decisionserver/decisionserverconsole/script $SCRIPT
COPY $ODMDOCKERDIR/decisionserver/decisionserverruntime/script $SCRIPT
COPY $ODMDOCKERDIR/licenses /licenses

# Install missing require package in the alpine builder image
RUN apk add --no-cache bash ca-certificates wget
# Build Welcome page
RUN cd /welcomepage; mvn -B clean install | grep -v 'Download.*' && mkdir -p $APPS

# Decision Server Console
COPY ./executionserver/applicationservers/WLP*/res.war $APPS
COPY ./executionserver/applicationservers/WLP*/DecisionService.war $APPS

RUN chmod -R a+x $SCRIPT && sync && if [ ! -f /config/resources/postgres* ]; then $SCRIPT/installPostgres.sh; fi

RUN $SCRIPT/extractApp.sh res.war && $SCRIPT/extractApp.sh DecisionService.war

RUN $SCRIPT/changeParamValue.sh management.protocol jmx tcpip $APPS/res.war/WEB-INF/web.xml && \
		$SCRIPT/changeParamValue.sh autoCreateSchema false true $APPS/res.war/WEB-INF/web.xml

RUN $SCRIPT/manageLicense.sh && $SCRIPT/loadFeatures.sh $SCRIPT && \
chmod -R 777 $APPS

FROM ${FROMLIBERTYBUILD} AS oidc-liberty-builder
ARG ODMDOCKERDIR
ENV ODMDOCKERDIR $ODMDOCKERDIR
USER root
ENV SCRIPT /script
COPY  $ODMDOCKERDIR/common/script $ODMDOCKERDIR/wlp* /opt/wlppackage/
RUN mkdir $SCRIPT && mv /opt/wlppackage/*.sh $SCRIPT && $SCRIPT/installFeatures.sh
COPY --chown=1001:0 $ODMDOCKERDIR/decisionserver/config /config
COPY --chown=1001:0 $ODMDOCKERDIR/decisionserver/decisionserverconsole/config /config
COPY --chown=1001:0 $ODMDOCKERDIR/common/config /config
COPY --chown=1001:0 $ODMDOCKERDIR/common/security/ltpa.keys /config/resources/security/ltpa.keys
COPY --chown=1001:0 $ODMDOCKERDIR/common/security/keystore.jks /config/security/keystore.jks
COPY --chown=1001:0 $ODMDOCKERDIR/common/security/truststore.jks /config/security/truststore.jks
COPY --chown=1001:0 $ODMDOCKERDIR/common/resources/ibm-docs.crt /config/resources/ibm-docs.crt
COPY --chown=1001:0 $ODMDOCKERDIR/common/drivers /config/resources

COPY --chown=1001:0 $ODMDOCKERDIR/common/config/jvm/jvm.options /config/configDropins/overrides/jvm.options
RUN chmod -R 777 /config

FROM ${FROMLIBERTY}
ARG ODMDOCKERDIR
ARG ODMVERSION
LABEL maintainer="ODMDev odmdev_open_source_user@wwpdl.vnet.ibm.com, Laurent GRATEAU <laurent.grateau@fr.ibm.com>"
MAINTAINER ODMDev odmdev_open_source_user@wwpdl.vnet.ibm.com, Laurent GRATEAU <laurent.grateau@fr.ibm.com>
LABEL name="Decision Server Console"
LABEL description="The Decision Server console provides a web-based graphical user interface to \
 manage and monitor RuleApps, ruleset archives, JavaPrr XOMs and libraries, and decision services."
LABEL vendor="IBM"
LABEL summary="A web-based graphical user interface to manage Decision Server artifacts."
LABEL version=${ODMVERSION}
LABEL ProductVersion=${ODMVERSION}
LABEL ProductID="IBM Operational Decision Manager for production"
LABEL io.openshift.tags="odm,dba,dbamc"
LABEL io.k8s.description="A web-based graphical user interface to manage Decision Server artifacts."
LABEL io.k8s.display-name="Decision Server Console"

ENV APP_NAME DecisionServerConsole
ENV ODMDOCKERDIR $ODMDOCKERDIR
ENV SCRIPT /script
ENV APPS /config/apps
ENV CONNECTION_POOL_SIZE 20

COPY --from=oidc-liberty-builder /opt/ibm/wlp /opt/ibm/wlp

# Welcome page
COPY --from=builder --chown=1001:0 /welcomepage/target/welcomepage.war $APPS
COPY --from=builder --chown=1001:0 $APPS $APPS
COPY --from=builder --chown=1001:0 /config/resources/postgres*  /config/resources
COPY --from=builder --chown=1001:0 $SCRIPT $SCRIPT

# License
COPY --from=builder  --chown=1001:0 /licenses /licenses
USER 1001
EXPOSE 9080 9443 1883

CMD ["/script/run.sh"]
