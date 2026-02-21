$TTL 3600
@	IN SOA dns200.anycast.me. tech.ovh.net. (2026021800 86400 3600 3600000 60)
        IN NS     dns200.anycast.me.
        IN NS     ns200.anycast.me.
        IN MX     10 mx1.improvmx.com.
        IN MX     20 mx2.improvmx.com.
        IN A     72.62.16.248
    600 IN TXT     "v=spf1 a mx include:spf.improvmx.com include:_spf.mailtrap.live -all"
    600 IN TXT     "1|www.windshear-ahead.com"
_acme-challenge        IN TXT     "qgTTWDDVFtyxcSkEa23t1TAqEqBrrOPMh8kzcYvBYIM"
_dmarc        IN TXT     "v=DMARC1; p=none; rua=mailto:dmarc@smtp-staging.mailtrap.net; ruf=mailto:dmarc@smtp-staging.mailtrap.net; rf=afrf; pct=100"
flightdeck777        IN A     72.62.16.248
ftp        IN CNAME     windshear-ahead.com.
mt-link        IN CNAME     t.mailtrap.live.
mt95        IN CNAME     smtp.mailtrap.live.
radar        IN A     72.62.16.248
rwmt1._domainkey        IN CNAME     rwmt1.dkim.smtp.mailtrap.live.
rwmt2._domainkey        IN CNAME     rwmt2.dkim.smtp.mailtrap.live.
www        IN A     72.62.16.248
www        IN TXT     "3|welcome"
www.radar        IN A     72.62.16.248
y01p3d-5k00h83w        IN A     72.62.16.248