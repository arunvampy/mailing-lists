# Set mailing list e-mail headers with Milter
This wiki page shows how to work around the problem of mail going to junk/spam folder for external recipients of Zimbra distribution lists.

By default Zimbra distribution lists do not set appropriate e-mail headers. This wiki page shows how to set-up Milter (a Python based extension for Postfix). Basically what Milter is set-up to do is filter the original message and discard it. Before discarding, it takes some headers and the body from the original mail and place them in a new mail and use sendmail to pass it to the DL.

The result is a clean message from the correct domain, and it should be able to pass SPF/DKIM and all that.
 
Its great, because the final message that goes to the recipients is still processed by Zimbra as a normal DL message.

On the CLI:

     yum install epel-release
     yum install python-pip supervisor python-pymilter sendmail
     pip install --upgrade pip     
     pip install python-libmilter
     
     systemctl disable sendmail
     systemctl stop sendmail
     systemctl mask sendmail

     mkdir - p /opt/zimbra_mailinglists_milter
     cd /opt/zimbra_mailinglists_milter
     wget https://raw.githubusercontent.com/Zimbra-Community/mailing-lists/master/milter/zimbra_mailinglists_milter.py -O /opt/zimbra_mailinglists_milter/zimbra_mailinglists_milter.py
     wget https://raw.githubusercontent.com/Zimbra-Community/mailing-lists/master/milter/etc/daemon.ini -O /etc/supervisord.d/zimbra_mailinglists_milter.ini

Then use your favorite editor (nano/vim) and open `/opt/zimbra_mailinglists_milter/zimbra_mailinglists_milter.py` look under `def eob(self , cmdDict)` and change the script to fit your needs. Please be aware that the indentation level of the statements is significant to Python.

While Zimbra comes with a built-in sendmail, it's configuration cannot be altered, so we use sendmail from the OS/distro. Configure it by setting 127.0.0.1 after DS to set Zimbra as the Smart Relay for the OS/distro:

     nano /etc/mail/sendmail.cf
     # "Smart" relay host (may be null)
     DS127.0.0.1

Enable and start the service.

     chmod +rx /opt/zimbra_mailinglists_milter/zimbra_mailinglists_milter.py
     systemctl start supervisord 
     systemctl enable supervisord
     tail -f /var/log/supervisor/supervisord.log
     netstat -tulpn | grep 5000 #should show the service

If you made any typos, you will see them in the log, and there will be nothing listening on port 5000. Try again and issue `systemctl stop supervisord && systemctl start supervisord`.

If it works, enable it for Zimbra:

     su - zimbra
     zmprov ms `zmhostname` zimbraMtaSmtpdMilters inet:127.0.0.1:5000
     zmprov ms `zmhostname` zimbraMtaNonSmtpdMilters inet:127.0.0.1:5000
     zmprov ms `zmhostname` zimbraMilterServerEnabled TRUE
     zmmtactl restart

     postconf smtpd_milters
     smtpd_milters = inet:127.0.0.1:5000, inet:127.0.0.1:7026

Try sending some emails and:

     tail -f /var/log/zimbra_mailinglists_milter.log
