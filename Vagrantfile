Vagrant.configure("2") do |config|

  config.vm.define :ldap do |ldap|
    ldap.vm.box = "ubuntu/trusty64"
    ldap.vm.hostname = "ldap"
    ldap.vm.network "private_network", ip: '10.0.0.3'
    ldap.vm.provision "shell", inline: <<-SHELL
      install -o root -g root -m 0644 /vagrant/common/hosts /etc/hosts

      ######################################
      # krb5
      ######################################
      debconf-set-selections /vagrant/krb5/krb5.debconf
      apt-get install -y krb5-kdc krb5-admin-server

      # creating the database from scratch every time takes too long
      # since there is no entropy yet
      # kdb5_util -r 'EXAMPLE.ORG' create -s -P 'secret'

      base64 -d < /vagrant/krb5/stash > /etc/krb5kdc/stash
      chmod 0400 /etc/krb5kdc/stash
      kdb5_util load /vagrant/krb5/krb5.dump
      install -m 0644 -o root -g root /vagrant/krb5/kdc.conf /etc/krb5kdc/kdc.conf
      install -m 0644 -o root -g root /vagrant/krb5/krb5.conf /etc/krb5.conf
      install -m 0644 -o root -g root /vagrant/krb5/kadm5.acl /etc/krb5kdc/kadm5.acl
      service krb5-kdc start
      service krb5-admin-server start

      ######################################
      # SASL
      ######################################
      apt-get install -y sasl2-bin libsasl2-modules-gssapi-mit
      install -m 0644 -o root -g root /vagrant/SASL/saslauthd /etc/default/saslauthd-slapd
      service saslauthd start
      # for debugging, export KRB5_TRACE=/dev/stderr

      kadmin.local -q 'addprinc -pw test user'

      # hostname has to be canonicalized to lowercase!
      kadmin.local -q "addprinc -randkey host/${HOSTNAME,,*}"
      kadmin.local -q "ktadd -k /etc/krb5.keytab -norandkey host/${HOSTNAME,,*}"

      testsaslauthd -f /var/run/slapd/sasl/mux -u user -p test

      ######################################
      # LDAP
      ######################################
      debconf-set-selections < /vagrant/LDAP/slapd.debconf
      apt-get install -y slapd ldap-utils
      install -d -m 0750 -o openldap -g openldap /etc/ldap/slapd.d
      sudo -u openldap slaptest -f /vagrant/LDAP/master-slapd.conf -F /etc/ldap/slapd.d
      install -d -m 0755 -o openldap -g openldap /var/run/slapd
      apt-get install -y slapd

      install -m 444 -o openldap -g openldap /vagrant/LDAP/ldap.conf /etc/ldap/ldap.conf

      ldapadd -f /etc/ldap/schema/cosine.ldif
      ldapadd -f /etc/ldap/schema/nis.ldif
      # the rest of UofA schema - optional for many applications
      ldapadd -f /vagrant/LDAP/uofa_schema.ldif

      ldapadd -f /vagrant/LDAP/populate.ldif

      # LDAP SASL binding config
      install -m 0644 -o root -g root /vagrant/LDAP/slapd.sasl.conf /etc/ldap/sasl2/slapd.conf
      usermod -a -G sasl openldap
      service slapd restart

      ldapsearch -x -b '' -s base supportedSASLmechanisms
      # should have 'EXTENAL', 'GSSAPI', and 'PLAIN' at least

      # ldap logging
      install -m 0444 -o root -g root /vagrant/LDAP/slapd.syslog.conf /etc/rsyslog.d/10-slapd.conf
      service rsyslog restart

      # temp fix: switch aa to complain mode for sasl to work
      apt-get install -y apparmor-utils
      aa-complain slapd

      ldapwhoami -x -D 'uid=user,dc=example,dc=ca' -w test
      # should succeed

      # ACLs/ID maps
      ldapmodify -f /vagrant/LDAP/acls.ldif

    SHELL
  end


  config.vm.define :mx do |mx|
    mx.vm.box = "ubuntu/xenial64"
    mx.vm.hostname = "mx"
    mx.vm.network "private_network", ip: '10.0.0.4'
    mx.vm.provision "shell", inline: <<-SHELL
      ######################################
      # LDAP
      ######################################
      debconf-set-selections < /vagrant/LDAP/slapd.debconf
      apt-get install -y slapd ldap-utils
      install -d -m 0750 -o openldap -g openldap /etc/ldap/slapd.d
      sudo -u openldap slaptest -f /vagrant/LDAP/slapd.conf -F /etc/ldap/slapd.d
      install -d -m 0755 -o openldap -g openldap /var/run/slapd
      apt-get install -y slapd

      install -m 444 -o openldap -g openldap /vagrant/LDAP/ldap.conf /etc/ldap/ldap.conf

      ldapadd -f /etc/ldap/schema/cosine.ldif
      ldapadd -f /etc/ldap/schema/nis.ldif
      # the rest of UofA schema - optional for many applications
      ldapadd -f /vagrant/LDAP/uofa_schema.ldif

      ldapadd -f /vagrant/LDAP/populate.ldif

     SHELL
  end

end
