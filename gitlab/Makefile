# New ports collection makefile for:   gitlab
# Whom:		Mikhail Krasilnikov <mk@dvaslona.ru>
#
# $FreeBSD$

PORTNAME=	gitlab
PORTVERSION=	4.2.0
CATEGORIES=	devel

MAINTAINER=	mk@dvaslona.ru
COMMENT=	Self hosted Git management software

USE_GITHUB=	yes
GH_ACCOUNT=	gitlabhq
GH_PROJECT=	gitlabhq
GH_TAGNAME=	v${PORTVERSION}
GH_COMMIT=	012fb5b

BUILD_DEPENDS+=	sudo:${PORTSDIR}/security/sudo

RUN_DEPENDS+=	redis-server:${PORTSDIR}/databases/redis \
		git:${PORTSDIR}/devel/git

USE_PYTHON=	2.5-2.9
USE_MYSQL=	server

GITLAB_USER=	gitlab
GITLAB_GROUP=	git
GITLAB_MYSQL_DB=	gitlab
GITLAB_MYSQL_USER=	${GITLAB_USER}
GITLAB_MYSQL_PASS=	gitlab

GIT_HOME=	${PREFIX}/git

GEMS_VERSION=	1.8.25
GEMS_DISTNAME=	rubygems-${GEMS_VERSION}
GEMS_FILENAME=	${GEMS_DISTNAME}.tgz

USE_RC_SUBR=	gitlab

SUB_FILES=	gitlab pkg-deinstall gitlab_install.sh mysql_setup.sql nginx.example apache.example
SUB_LIST=	GITLAB_USER=${GITLAB_USER} \
		GIT_HOME=${GIT_HOME} \
		GITLAB_MYSQL_DB=${GITLAB_MYSQL_DB} \
		GITLAB_MYSQL_USER=${GITLAB_MYSQL_USER} \
		GITLAB_MYSQL_PASS=${GITLAB_MYSQL_PASS}

USERS=		git
GROUPS=		${GITLAB_GROUP}


.include <bsd.port.pre.mk>


do-build:
	@${CP} ${WRKSRC}/config/gitlab.yml.example ${WRKSRC}/config/gitlab.yml
	@${REINPLACE_CMD} -e 's|/home/git/|${GIT_HOME}/|g' ${WRKSRC}/config/gitlab.yml
	@${CP} ${WRKSRC}/config/unicorn.rb.example ${WRKSRC}/config/unicorn.rb
	@${REINPLACE_CMD} -e 's|/home/gitlab/gitlab|${WWWDIR}|g' ${WRKSRC}/config/unicorn.rb
	@${CP} ${WRKSRC}/config/database.yml.mysql ${WRKSRC}/config/database.yml
	@${REINPLACE_CMD} -e 's|gitlabhq_production|${GITLAB_MYSQL_DB}|g' ${WRKSRC}/config/database.yml
	@${REINPLACE_CMD} -e 's|root|${GITLAB_MYSQL_USER}|g' ${WRKSRC}/config/database.yml
	@${REINPLACE_CMD} -e 's|secure password|${GITLAB_MYSQL_PASS}|g' ${WRKSRC}/config/database.yml
	@if ! which -s gem19; then \
		if [ ! -e ${DISTDIR}/${GEMS_FILENAME} ]; then \
			(cd ${DISTDIR} && ${FETCH_CMD} http://rubyforge.org/frs/download.php/76729/${GEMS_FILENAME}); \
		fi; \
		(cd ${WRKDIR} && tar xzf ${DISTDIR}/${GEMS_FILENAME}); \
	fi

do-install:
	@if ! pkg_info 'ruby=>1.9.3,1' 2>/dev/null; then \
		(cd ${PORTSDIR}/lang/ruby19 && make install clean); \
	fi
	@pw useradd ${GITLAB_USER} -g ${GITLAB_GROUP} -d ${WWWDIR} -m -s /bin/nologin -c 'GitLab'
	@(cd ${WRKSRC} && ${COPYTREE_SHARE} \* ${WWWDIR})
	@${CHOWN} -R ${GITLAB_USER}:${GITLAB_GROUP} ${WWWDIR}
	@sudo -u ${GITLAB_USER} -H ssh-keygen -q -N '' -t rsa -f ${WWWDIR}/.ssh/id_rsa
	@${MKDIR} ${GIT_HOME}
	@${CHOWN} git:git ${GIT_HOME}
	@(cd ${GIT_HOME} && sudo -u git -H git clone -b gl-v320 https://github.com/gitlabhq/gitolite.git gitolite)
	@sudo -u git -H ${MKDIR} ${GIT_HOME}/bin
	@sudo -u git -H sh -c echo "PATH=$$PATH:${GIT_HOME}/bin" >> ${GIT_HOME}/.profile
	@sudo -u git -H sh -c echo "export PATH" >> ${GIT_HOME}/.profile
	@sudo -u git -H sh -c '${GIT_HOME}/gitolite/install -ln ${GIT_HOME}/bin'
	@cp ${WWWDIR}/.ssh/id_rsa.pub ${GIT_HOME}/gitlab.pub
	@${CHMOD} 0444 ${GIT_HOME}/gitlab.pub
	@sudo -u git -H sh -c "PATH=${GIT_HOME}/bin:$$PATH; gitolite setup -pk ${GIT_HOME}/gitlab.pub"
	@${MKDIR} -m 750 ${GIT_HOME}/.gitolite
	@${CHOWN} -R git:git ${GIT_HOME}
	@${CHMOD} -R ug+rwX,o-rwx ${GIT_HOME}/repositories/
	@${CHOWN} -R git:git ${GIT_HOME}/repositories/
	@sudo -u git -H find ${GIT_HOME}/repositories -type d -print0 | xargs -0 chmod g+s
	@sudo -u ${GITLAB_USER} -H ssh-keyscan -t rsa,dsa localhost >> ${WWWDIR}/.ssh/known_hosts
	@sudo -u ${GITLAB_USER} -H ssh-keyscan -t rsa,dsa 127.0.0.1 >> ${WWWDIR}/.ssh/known_hosts
	@${CHOWN} -R ${GITLAB_USER} ${WWWDIR}/log ${WWWDIR}/tmp
	@${CHMOD} -R u+rwX  ${WWWDIR}/log ${WWWDIR}/tmp
	@sudo -u ${GITLAB_USER} -H ${MKDIR} ${WWWDIR}/gitlab-satellites
	@if ! which -s gem19; then \
		(cd ${WRKDIR}/${GEMS_DISTNAME} && ruby19 setup.rb --prefix ${PREFIX}); \
	fi
	@gem19 install bundler
	@gem19 install charlock_holmes --version '0.6.9'
	@(cd ${WWWDIR} && sudo -u ${GITLAB_USER} -H bundle install --deployment --without development test postgres)
	@${MKDIR} ${WWWDIR}/distrib
	@${CP} ${WRKDIR}/mysql_setup.sql ${WWWDIR}/distrib/
	@${CP} ${WRKDIR}/nginx.example ${WWWDIR}/distrib/
	@${CP} ${WRKDIR}/apache.example ${WWWDIR}/distrib/
	@${INSTALL_SCRIPT} ${WRKDIR}/gitlab_install.sh ${WWWDIR}/distrib/

post-install:
	@${CAT} ${PKGMESSAGE} | sed -r 's|%%WWWDIR%%|${WWWDIR}|g' | sed -r 's|%%PREFIX%%|${LOCALBASE}|g'

.include <bsd.port.post.mk>
