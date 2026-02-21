# Complete Email Setup Guide: ImprovMX + Mailtrap with CloudPanel

This guide covers the complete setup of a professional email infrastructure for your domains hosted on CloudPanel, using ImprovMX for receiving emails and Mailtrap for sending from Laravel applications.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         INCOMING EMAILS                                  │
│                                                                          │
│   someone@example.com ──► contact@windshear-ahead.com                   │
│                                    │                                     │
│                                    ▼                                     │
│                          ┌─────────────────┐                            │
│                          │    ImprovMX     │                            │
│                          │   (MX Records)  │                            │
│                          └────────┬────────┘                            │
│                                   │                                      │
│                                   ▼                                      │
│                          your.gmail@gmail.com                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                         OUTGOING EMAILS                                  │
│                                                                          │
│   Laravel App (notifications, confirmations, etc.)                      │
│         │                                                                │
│         ▼                                                                │
│   ┌───────────┐         ┌─────────────────┐         ┌─────────────┐    │
│   │  .env     │ ──────► │    Mailtrap     │ ──────► │  Recipient  │    │
│   │  SMTP     │         │  SMTP Relay     │         │   Inbox     │    │
│   └───────────┘         └─────────────────┘         └─────────────┘    │
│                                                                          │
│   Gmail "Send as" (manual replies)                                      │
│         │                                                                │
│         ▼                                                                │
│   ┌───────────┐         ┌─────────────────┐         ┌─────────────┐    │
│   │  Gmail    │ ──────► │    Mailtrap     │ ──────► │  Recipient  │    │
│   │  SMTP     │         │  SMTP Relay     │         │   Inbox     │    │
│   └───────────┘         └─────────────────┘         └─────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: ImprovMX Setup (Receiving Emails)

ImprovMX handles all incoming emails and forwards them to your Gmail account.

### 1.1 Create ImprovMX Account

1. Go to [https://improvmx.com](https://improvmx.com)
2. Sign up for a free account
3. Verify your email address

### 1.2 Add Your Domain

1. In ImprovMX dashboard, click **"Add Domain"**
2. Enter your domain (e.g., `windshear-ahead.com`)
3. Click **"Add"**

### 1.3 Create Email Aliases

1. Go to your domain settings in ImprovMX
2. Add aliases (forwarding rules):

| Alias | Forwards To |
|-------|-------------|
| `contact` | your.gmail@gmail.com |
| `support` | your.gmail@gmail.com |
| `*` (catch-all) | your.gmail@gmail.com |

> **Tip**: The catch-all (`*`) alias forwards any address that doesn't match a specific alias.

### 1.4 Configure DNS Records

Access your DNS management (at your registrar or DNS provider like OVH).

#### MX Records (Mail Exchange)

Remove any existing MX records, then add:

| Type | Name | Priority | Value |
|------|------|----------|-------|
| MX | @ | 10 | mx1.improvmx.com |
| MX | @ | 20 | mx2.improvmx.com |

#### SPF Record

| Type | Name | Value |
|------|------|-------|
| TXT | @ | `v=spf1 include:spf.improvmx.com include:_spf.mailtrap.live -all` |

> **Note**: This SPF record already includes both ImprovMX (receiving) and Mailtrap (sending).

#### DMARC Record

| Type | Name | Value |
|------|------|-------|
| TXT | `_dmarc` | `v=DMARC1; p=quarantine; rua=mailto:your.gmail@gmail.com` |

> **Note**: DKIM is managed directly by ImprovMX and Mailtrap. No DNS records are required for DKIM.

### 1.5 Verify DNS Configuration

1. Return to ImprovMX dashboard
2. Go to your domain settings (cogwheel icon) → **DNS Settings**
3. Click **"Check DNS"** or wait for automatic verification
4. All records should show green checkmarks

### 1.6 Test Incoming Email

Send a test email from an external account to `contact@yourdomain.com`. It should arrive in your Gmail inbox within a few minutes.

---

## Part 2: Mailtrap Setup (Sending Emails)

Mailtrap handles all outgoing emails from your Laravel applications.

### 2.1 Create/Access Mailtrap Account

1. Go to [https://mailtrap.io](https://mailtrap.io)
2. Sign up or log in to your existing account
3. Navigate to **"Email API/SMTP"** → **"Sending Domains"**

### 2.2 Add Your Sending Domain

1. Click **"Add Domain"**
2. Enter your domain (e.g., `windshear-ahead.com`)
3. Mailtrap will provide DNS records to verify ownership

### 2.3 Configure DNS Records for Mailtrap

The SPF record was already configured in Part 1.4. You only need to add the domain verification record that Mailtrap provides.

#### Domain Verification Record

Mailtrap will ask you to add a TXT record for domain verification. It looks like:

| Type | Name | Value |
|------|------|-------|
| TXT | @ or specific subdomain | (provided by Mailtrap dashboard) |

> **Note**: DKIM is managed directly by Mailtrap. No DKIM DNS records are required.

### 2.4 Verify Domain in Mailtrap

1. Return to Mailtrap dashboard
2. Click **"Verify DNS Records"**
3. Domain verification should show green checkmark

### 2.5 Get SMTP Credentials

1. In Mailtrap, go to **"Email API/SMTP"** → **"Sending Domains"**
2. Click on your verified domain
3. Go to **"Integration"** → **"SMTP"**
4. Note down:
   - **Host**: `live.smtp.mailtrap.io`
   - **Port**: `587`
   - **Username**: `api`
   - **Password**: Your API token (click to reveal/copy)

---

## Part 3: Laravel Configuration

### 3.1 Environment Configuration

Update your `.env` file:

```env
# Mail Configuration - Mailtrap Production SMTP
MAIL_MAILER=smtp
MAIL_HOST=live.smtp.mailtrap.io
MAIL_PORT=587
MAIL_USERNAME=api
MAIL_PASSWORD=your_mailtrap_api_token_here
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=contact@windshear-ahead.com
MAIL_FROM_NAME="${APP_NAME}"
```

### 3.2 Mail Configuration File

Ensure your `config/mail.php` is properly configured:

```php
<?php

return [
    'default' => env('MAIL_MAILER', 'smtp'),

    'mailers' => [
        'smtp' => [
            'transport' => 'smtp',
            'host' => env('MAIL_HOST', 'smtp.mailgun.org'),
            'port' => env('MAIL_PORT', 587),
            'encryption' => env('MAIL_ENCRYPTION', 'tls'),
            'username' => env('MAIL_USERNAME'),
            'password' => env('MAIL_PASSWORD'),
            'timeout' => null,
            'local_domain' => env('MAIL_EHLO_DOMAIN'),
        ],
    ],

    'from' => [
        'address' => env('MAIL_FROM_ADDRESS', 'hello@example.com'),
        'name' => env('MAIL_FROM_NAME', 'Example'),
    ],
];
```

### 3.3 Test Email Sending

Create a simple Artisan command to test:

```bash
php artisan make:command TestEmail
```

Edit `app/Console/Commands/TestEmail.php`:

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\Mail;

class TestEmail extends Command
{
    protected $signature = 'mail:test {email}';
    protected $description = 'Send a test email to verify SMTP configuration';

    public function handle(): int
    {
        $recipient = $this->argument('email');

        Mail::raw('This is a test email from your Laravel application.', function ($message) use ($recipient) {
            $message->to($recipient)
                    ->subject('Test Email - Laravel SMTP Configuration');
        });

        $this->info("Test email sent to {$recipient}");
        
        return Command::SUCCESS;
    }
}
```

Run the test:

```bash
php artisan mail:test your.gmail@gmail.com
```

### 3.4 Using Different From Addresses

If you need to send from different addresses (all must be on verified domains):

```php
use Illuminate\Support\Facades\Mail;

Mail::raw('Message content', function ($message) {
    $message->from('support@windshear-ahead.com', 'Support Team')
            ->to('recipient@example.com')
            ->subject('Support Ticket Response');
});
```

### 3.5 Mailable Example

Create a proper Mailable class:

```bash
php artisan make:mail WelcomeEmail
```

Edit `app/Mail/WelcomeEmail.php`:

```php
<?php

namespace App\Mail;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Mail\Mailable;
use Illuminate\Mail\Mailables\Content;
use Illuminate\Mail\Mailables\Envelope;
use Illuminate\Queue\SerializesModels;

class WelcomeEmail extends Mailable implements ShouldQueue
{
    use Queueable, SerializesModels;

    public function __construct(
        public string $userName,
    ) {}

    public function envelope(): Envelope
    {
        return new Envelope(
            subject: 'Welcome to Our Platform',
        );
    }

    public function content(): Content
    {
        return new Content(
            view: 'emails.welcome',
            with: [
                'userName' => $this->userName,
            ],
        );
    }

    public function attachments(): array
    {
        return [];
    }
}
```

---

## Part 4: Gmail "Send As" Configuration

This allows you to reply to emails from Gmail using your custom domain address.

### 4.1 Get Mailtrap SMTP Credentials for Gmail

Use the same credentials from Part 2.5:
- **SMTP Server**: `live.smtp.mailtrap.io`
- **Port**: `587`
- **Username**: `api`
- **Password**: Your Mailtrap API token

### 4.2 Configure Gmail

1. Open Gmail → **Settings** (gear icon) → **See all settings**
2. Go to **"Accounts and Import"** tab
3. Find **"Send mail as"** section
4. Click **"Add another email address"**

5. In the popup:
   - **Name**: Your display name (e.g., "Joey - Windshear Ahead")
   - **Email address**: `contact@windshear-ahead.com`
   - Uncheck "Treat as an alias" if you want separate reply-to handling
   - Click **"Next Step"**

6. SMTP Server configuration:
   - **SMTP Server**: `live.smtp.mailtrap.io`
   - **Port**: `587`
   - **Username**: `api`
   - **Password**: Your Mailtrap API token
   - Select **"Secured connection using TLS"**
   - Click **"Add Account"**

7. Gmail will send a verification email to confirm ownership. Since ImprovMX forwards to your Gmail, you'll receive it and can click the verification link.

### 4.3 Set as Default (Optional)

If you want `contact@windshear-ahead.com` as your default sending address:

1. In Gmail Settings → **"Accounts and Import"**
2. Under **"Send mail as"**, click **"make default"** next to your domain address

### 4.4 Reply Behavior

Configure how Gmail handles replies:

1. In **"Accounts and Import"**
2. Under **"Send mail as"**, choose:
   - **"Reply from the same address the message was sent to"** (recommended)
   
This ensures replies to `contact@windshear-ahead.com` automatically use that address.

---

## Part 5: DNS Records Summary

Here's a complete summary of all DNS records for a domain (e.g., `windshear-ahead.com`):

### MX Records

| Priority | Value |
|----------|-------|
| 10 | mx1.improvmx.com |
| 20 | mx2.improvmx.com |

### TXT Records

| Name | Value |
|------|-------|
| @ | `v=spf1 include:spf.improvmx.com include:_spf.mailtrap.live -all` |
| _dmarc | `v=DMARC1; p=quarantine; rua=mailto:your.gmail@gmail.com` |

### Domain Verification (Mailtrap)

| Name | Value |
|------|-------|
| @ or subdomain | (provided by Mailtrap dashboard) |

> **Note**: No DKIM records required. DKIM is managed directly by ImprovMX and Mailtrap.

---

## Part 6: Testing & Verification

### 6.1 DNS Propagation Check

Use these tools to verify DNS propagation:

```bash
# Check MX records
dig MX windshear-ahead.com +short

# Check SPF
dig TXT windshear-ahead.com +short

# Check DMARC
dig TXT _dmarc.windshear-ahead.com +short
```

Or use online tools:
- [MXToolbox](https://mxtoolbox.com/)
- [Mail-tester](https://www.mail-tester.com/) (send an email to their test address)
- [DMARC Analyzer](https://www.dmarcanalyzer.com/dmarc/dmarc-record-check/)

### 6.2 Email Deliverability Test

1. Go to [https://www.mail-tester.com](https://www.mail-tester.com)
2. Copy the unique email address provided
3. Send an email to that address from your Laravel app:

```bash
php artisan mail:test unique-address@srv1.mail-tester.com
```

4. Check your score (aim for 9/10 or higher)

### 6.3 Verify Incoming Email Flow

1. Send an email from an external account (e.g., personal Gmail) to `test@windshear-ahead.com`
2. Verify it arrives in your configured Gmail inbox
3. Check headers to confirm ImprovMX processed it

### 6.4 Verify Outgoing Email Flow

1. From Gmail, compose a new email
2. In the "From" dropdown, select your domain address
3. Send to an external account
4. Check the email headers to verify:
   - SPF: PASS
   - DKIM: PASS
   - DMARC: PASS

---

## Part 7: Multiple Domains Setup

Repeat the process for each additional domain (e.g., `lambda-aero.com`).

### 7.1 ImprovMX

1. Add domain in ImprovMX dashboard
2. Configure DNS (MX records)
3. Set up aliases/forwarding rules
4. Update SPF to include ImprovMX

### 7.2 Mailtrap

1. Add domain in Mailtrap dashboard
2. Add domain verification TXT record
3. SPF should already include Mailtrap

### 7.3 Laravel Multi-Domain Support

For different "from" addresses per domain, you can configure dynamically:

```php
// In a service provider or controller
config(['mail.from.address' => 'contact@lambda-aero.com']);
config(['mail.from.name' => 'Lambda Aero']);

// Or per-email basis
Mail::raw('Content', function ($message) {
    $message->from('contact@lambda-aero.com', 'Lambda Aero')
            ->to('recipient@example.com');
});
```

---

## Part 8: Monitoring & Troubleshooting

### 8.1 Mailtrap Dashboard

Monitor your email sending:
- **Email logs**: View sent emails, delivery status
- **Analytics**: Open rates, click rates, bounces
- **Suppressions**: Manage bounced/unsubscribed addresses

### 8.2 ImprovMX Dashboard

Monitor incoming email:
- **Logs**: View forwarded emails (Premium feature)
- **Aliases**: Manage forwarding rules
- **Domain settings**: Check DNS status

### 8.3 Common Issues

**Emails going to spam:**
- Verify SPF and DMARC are passing
- Check IP reputation at [MXToolbox Blacklist Check](https://mxtoolbox.com/blacklists.aspx)
- Ensure proper "From" address formatting

**Emails not being received:**
- Check MX records propagation
- Verify ImprovMX alias configuration
- Check spam folder

**Laravel not sending:**
- Clear config cache: `php artisan config:clear`
- Verify `.env` credentials
- Check Laravel logs: `storage/logs/laravel.log`

**Gmail "Send As" verification not received:**
- Check ImprovMX alias is properly configured
- Check spam folder
- Try resending verification

### 8.4 Useful Debug Commands

```bash
# Test SMTP connection from server
openssl s_client -starttls smtp -connect live.smtp.mailtrap.io:587

# Send test email via command line (with swaks if installed)
swaks --to test@example.com \
      --from contact@windshear-ahead.com \
      --server live.smtp.mailtrap.io:587 \
      --auth LOGIN \
      --auth-user api \
      --auth-password "your_token" \
      --tls

# Check DNS records
dig ANY windshear-ahead.com
```

---

## Quick Reference Card

### ImprovMX (Receiving)
- **Website**: improvmx.com
- **MX1**: mx1.improvmx.com (priority 10)
- **MX2**: mx2.improvmx.com (priority 20)
- **SPF include**: `include:spf.improvmx.com`
- **DKIM**: Managed automatically (no DNS required)

### Mailtrap (Sending)
- **Website**: mailtrap.io
- **SMTP Host**: live.smtp.mailtrap.io
- **SMTP Port**: 587 (TLS)
- **Username**: api
- **Password**: Your API token
- **SPF include**: `include:_spf.mailtrap.live`
- **DKIM**: Managed automatically (no DNS required)

### Combined SPF Record
```
v=spf1 include:spf.improvmx.com include:_spf.mailtrap.live -all
```

### Free Tier Limits
- **ImprovMX**: 25 aliases per domain, unlimited forwarding
- **Mailtrap**: 1,000 emails/month

---

*Guide created for CloudPanel environment - December 2024*
