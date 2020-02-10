# createT3Setup

#!/bin/bash

pathDir="/var/www/"
pathApacheAvailable="/etc/apache2/sites-available/"
pathApacheEnabled="/etc/apache2/sites-enabled/"
pathTYPO3="/var/www/TYPO3/"
pathTYPO3Version="/var/www/TYPO3/version/"
domainExt="test"
versionTYPO3="8.7.9"
userPermission="vagrant"
groupPermission="vagrant"

usage() { echo "Usage: $0 -n <siteName> [-h /<homeDir>/] [-e <domainExt] [-v <versionTYPO3> ] [-d <dbName>] [-u <dbName>] [-p <dbPass>]" 1>&2; exit 1; }

while getopts ":n:h:e:v:d:u:p" o; do
        case "${o}" in
                n)
                        n=${OPTARG}
                        ;;
                h)
                        h=${OPTARG}
                        ;;
                e)
                        e=${OPTARG}
                        ;;
                v)
                        v=${OPTARG}
                        ;;
                d)
                        d=${OPTARG}
                        ;;
                u)
                        u=${OPTARG}
                        ;;
                p)
                        p=${OPTARG}
                        ;;
        esac
done
shift $((OPTIND-1))
if [ -z "${n}" ]; then
        usage
fi
if [ -z "${h}" ]; then
        h=${pathDir}
fi
if [ -z "${e}" ]; then
        e=${domainExt}
fi
if [ -z "${v}" ]; then
        v=${versionTYPO3}
fi
if [ -z "${p}" ]; then
        p="$(openssl rand -base64 12)"
fi
if [ -z "${d}" ]; then
        d="typo3_${n}"
fi
if [ -z "${u}" ]; then
        u=${n}
fi

mkdir -p ${h}${n}.${e}
echo "Created directory ${h}${n}.${e}";
mkdir -p ${h}${n}.${e}/www/
echo "Created directory ${h}${n}.${e}/www/";
mkdir -p ${h}${n}.${e}/log/
echo "Created directory ${h}${n}.${e}/log/";
mkdir -p ${h}${n}.${e}/conf/
echo "Created directory ${h}${n}.${e}/conf/";

ln -sf ${pathTYPO3Version}typo3_src-${versionTYPO3} ${pathTYPO3}${n}.${e}
echo "Created link ${pathTYPO3}${n}.${e}"
ln -sf ${pathTYPO3}${n}.${e} ${h}${n}.${e}/www/typo3_src
echo "Created link ${h}${n}.${e}/www/typo3_src"
cd ${h}${n}.${e}/www/
ln -sf typo3_src/typo3
echo "Created link ${h}${n}.${e}/www/typo3"
ln -sf typo3_src/index.php
echo "Created link ${h}${n}.${e}/www/index.php"
cp ${h}${n}.${e}/www/typo3_src/_.htaccess ${h}${n}.${e}/www/.htaccess
echo "Copied file ${h}${n}.${e}/www/.htaccess"
touch "${h}${n}.${e}/www/FIRST_INSTALL"
echo "Created file FIRST_INSTALL";

chown ${userPermission}:${groupPermission} -R ${h}${n}.${e}/www/
chown ${userPermission}:${groupPermission} -R ${h}${n}.${e}/log/
echo "Setted file permission"

confFile="${h}${n}.${e}/conf/apache-${n}_${e}.conf"
/bin/cat <<EOM >${confFile}
<VirtualHost *:80>
    DocumentRoot /var/www/${n}.${e}/www
    ServerName www.${n}.${e}
    ServerAlias ${n}.${e}
    #SetEnv TYPO3_CONTEXT Development
    LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined
    LogFormat "%h %l %u %t \"%r\" %>s %b" common
    LogFormat "%{Referer}i -> %U" referer
    LogFormat "%{User-agent}i" agent
    CustomLog /var/www/${n}.${e}/log/access.log common
    ErrorLog  /var/www/${n}.${e}/log/error.log
</VirtualHost>
EOM
echo "Created file ${confFile}"

ln -sf ${confFile} ${pathApacheAvailable}apache-${n}_${e}.conf
ln -sf ${pathApacheAvailable}apache-${n}_${e}.conf ${pathApacheEnabled}apache-${n}_${e}.conf
echo "Created link Apache ${pathApacheAvailable}apache-${n}_${e}.conf ${pathApacheEnabled}apache-${n}_${e}.conf"
service apache2 reload
echo "Reloaded apache2"

mysql -p -e "create database ${d} character set utf8"
mysql -p -e "grant all on ${d}.* to '${u}'@'localhost' identified by '${p}';"
echo "Created db ${d} to ${u}:${p}"
echo ""
echo "Connect now with browser to ${n}.${e}"
