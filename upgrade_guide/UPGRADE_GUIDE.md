# MessageMedia PHP SDK Refactor Guide
## Deprecation to Simple cURL REST API

**Document Version**: 1.0  
**Date**: December 2025  
**Original SDK**: [messagemedia/messages-php-sdk](https://github.com/messagemedia/messages-php-sdk)  
**API Docs**: [MessageMedia Messaging REST API](https://messagemedia.github.io/documentation/#tag/Messages)

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [What Changed](#what-changed)
3. [Migration Strategy](#migration-strategy)
4. [Installation](#installation)
5. [Configuration](#configuration)
6. [API Endpoints](#api-endpoints)
7. [Code Migration Examples](#code-migration-examples)
8. [Class Structure](#class-structure)
9. [Authentication](#authentication)
10. [Error Handling](#error-handling)
11. [Testing Migration](#testing-migration)
12. [Package Installation](#package-installation)

---

## Executive Summary

The MessageMedia PHP SDK has been **officially deprecated and is no longer maintained**. This guide provides a complete refactor strategy to migrate to direct REST API calls using simple cURL instead.

### Key Benefits of Migration

✅ **Lightweight**: Removes unnecessary SDK dependencies  
✅ **Direct API Access**: Full control over requests/responses  
✅ **Modern**: Uses PHP 8+ best practices  
✅ **Laravel Native**: Easy integration as a Laravel package  
✅ **Flexible**: Simple to extend and customize  
✅ **No Breaking Changes**: Same class objects returned (for responses & exceptions)

---

## What Changed

### Before (Deprecated SDK)

```php
$authUserName = 'API_KEY';
$authPassword = 'API_SECRET';
$useHmacAuthentication = false;

$client = new MessageMediaMessagesLib\MessageMediaMessagesClient(
    $authUserName, 
    $authPassword, 
    $useHmacAuthentication
);

$controller = $client->getMessages();

$body = new SendMessagesRequest();
$body->messages = [];

$message = new Message();
$message->content = 'Hello world!';
$message->destinationNumber = '+61491570156';
$body->messages[] = $message;

try {
    $result = $controller->sendMessages($body);
    echo $result;
} catch (Exception $e) {
    echo $e;
}
```

### After (cURL Wrapper)

```php
use Infoxchange\MessageMedia\Client;
use Infoxchange\MessageMedia\Request\SendMessagesRequest;
use Infoxchange\MessageMedia\Message;

$client = new Client(
    apiKey: 'API_KEY',
    apiSecret: 'API_SECRET'
);

$request = new SendMessagesRequest();

$message = new Message();
$message->content = 'Hello world!';
$message->destinationNumber = '+61491570156';
$request->messages = [$message];

try {
    $response = $client->sendMessages($request);
    echo $response;
} catch (Exception $e) {
    echo $e->getMessage();
}
```

**Result**: Same interface, cleaner implementation, zero external dependencies!

---

## Migration Strategy

### Phase 1: Preparation
- [ ] Audit current SDK usage in codebase
- [ ] Document all API endpoints being used
- [ ] Create comprehensive test suite
- [ ] Set up staging environment

### Phase 2: Implementation
- [ ] Install new package via Composer
- [ ] Register service provider in Laravel
- [ ] Publish configuration file
- [ ] Implement cURL wrapper classes

### Phase 3: Migration
- [ ] Update service/class imports
- [ ] Replace SDK client initialization
- [ ] Update request building (90% compatible)
- [ ] Test all messaging workflows

### Phase 4: Validation & Cleanup
- [ ] Run full test suite
- [ ] Test error handling
- [ ] Remove SDK dependencies
- [ ] Performance testing
- [ ] Deploy to production

---

## Installation

### 1. Via Composer (Future Package)

```bash
composer require infoxchange/messagemedia-laravel
```

### 2. Manual Installation

Copy the package files to `packages/messagemedia-laravel/` directory.

### 3. Register Service Provider

In `config/app.php`:

```php
'providers' => [
    // ...
    Infoxchange\MessageMedia\ServiceProvider::class,
],
```

Or use **auto-discovery** (Laravel 5.5+) - no registration needed!

---

## Configuration

### 1. Publish Configuration

```bash
php artisan vendor:publish --provider="Infoxchange\MessageMedia\ServiceProvider"
```

### 2. Set Environment Variables

```env
MESSAGEMEDIA_API_KEY=your_api_key_here
MESSAGEMEDIA_API_SECRET=your_api_secret_here
MESSAGEMEDIA_USE_HMAC=false
MESSAGEMEDIA_BASE_URL=https://api.messagemedia.com/v1
```

### 3. Configuration File (`config/messagemedia.php`)

```php
return [
    'api_key' => env('MESSAGEMEDIA_API_KEY'),
    'api_secret' => env('MESSAGEMEDIA_API_SECRET'),
    'use_hmac' => env('MESSAGEMEDIA_USE_HMAC', false),
    'base_url' => env('MESSAGEMEDIA_BASE_URL', 'https://api.messagemedia.com/v1'),
    'timeout' => env('MESSAGEMEDIA_TIMEOUT', 30),
    'verify_ssl' => env('MESSAGEMEDIA_VERIFY_SSL', true),
];
```

---

## API Endpoints

### Supported Endpoints

| Operation | Method | Endpoint | Class |
|-----------|--------|----------|-------|
| Send Messages | POST | `/messages/send` | `SendMessagesRequest` |
| Check Replies | GET | `/replies` | `CheckRepliesRequest` |
| Confirm Replies | PUT | `/replies/{replyId}` | `ConfirmRepliesRequest` |
| Get Delivery Reports | GET | `/delivery-reports` | `CheckDeliveryReportsRequest` |
| Confirm Delivery Reports | PUT | `/delivery-reports/{deliveryReportId}` | `ConfirmDeliveryReportsRequest` |

### Base URL

```
https://api.messagemedia.com/v1
```

### Authentication

**Type**: Basic Authentication  
**Header**: `Authorization: Basic base64(API_KEY:API_SECRET)`

---

## Code Migration Examples

### Example 1: Send Single Message

#### Before (Deprecated SDK)

```php
use MessageMediaMessagesLib\MessageMediaMessagesClient;
use MessageMediaMessagesLib\Models\SendMessagesRequest;
use MessageMediaMessagesLib\Models\Message;

$client = new MessageMediaMessagesClient('API_KEY', 'API_SECRET');
$controller = $client->getMessages();

$body = new SendMessagesRequest();
$body->messages = [];

$message = new Message();
$message->content = 'Hello World!';
$message->destinationNumber = '+61491570156';
$body->messages[] = $message;

try {
    $result = $controller->sendMessages($body);
    echo $result->messages[0]->messageId;
} catch (Exception $e) {
    echo $e->getMessage();
}
```

#### After (cURL Implementation)

```php
use Infoxchange\MessageMedia\Client;
use Infoxchange\MessageMedia\Request\SendMessagesRequest;
use Infoxchange\MessageMedia\Message;

$client = new Client(
    apiKey: config('messagemedia.api_key'),
    apiSecret: config('messagemedia.api_secret')
);

$request = new SendMessagesRequest();

$message = new Message();
$message->content = 'Hello World!';
$message->destinationNumber = '+61491570156';
$request->messages = [$message];

try {
    $response = $client->sendMessages($request);
    echo $response->messages[0]->messageId;
} catch (Exception $e) {
    echo $e->getMessage();
}
```

---

### Example 2: Send Multiple Messages with Metadata

#### Before

```php
$body = new SendMessagesRequest();
$body->messages = [];

for ($i = 0; $i < 2; $i++) {
    $message = new Message();
    $message->content = "Message $i";
    $message->destinationNumber = "+6149157015{$i}";
    $message->metadata = ['order_id' => "ORD-{$i}"];
    $message->callbackUrl = 'https://example.com/callback';
    $body->messages[] = $message;
}

$result = $controller->sendMessages($body);
```

#### After

```php
$request = new SendMessagesRequest();

for ($i = 0; $i < 2; $i++) {
    $message = new Message();
    $message->content = "Message $i";
    $message->destinationNumber = "+6149157015{$i}";
    $message->metadata = ['order_id' => "ORD-{$i}"];
    $message->callbackUrl = 'https://example.com/callback';
    $request->messages[] = $message;
}

$response = $client->sendMessages($request);
```

---

### Example 3: Check Replies

#### Before

```php
$checkRepliesRequest = new CheckRepliesRequest();
$checkRepliesRequest->limit = 10;

$result = $controller->getCheckReplies($checkRepliesRequest);

foreach ($result->replies as $reply) {
    echo $reply->replyId . ": " . $reply->content;
}
```

#### After

```php
use Infoxchange\MessageMedia\Request\CheckRepliesRequest;

$request = new CheckRepliesRequest();
$request->limit = 10;

$response = $client->checkReplies($request);

foreach ($response->replies as $reply) {
    echo $reply->replyId . ": " . $reply->content;
}
```

---

### Example 4: Get Delivery Reports

#### Before

```php
$checkDeliveryReportsRequest = new CheckDeliveryReportsRequest();
$checkDeliveryReportsRequest->limit = 10;

$result = $controller->getCheckDeliveryReports($checkDeliveryReportsRequest);

foreach ($result->deliveryReports as $report) {
    echo $report->messageId . ": " . $report->status;
}
```

#### After

```php
use Infoxchange\MessageMedia\Request\CheckDeliveryReportsRequest;

$request = new CheckDeliveryReportsRequest();
$request->limit = 10;

$response = $client->checkDeliveryReports($request);

foreach ($response->deliveryReports as $report) {
    echo $report->messageId . ": " . $report->status;
}
```

---

## Class Structure

### Core Classes Maintained

The refactored implementation maintains all request/response classes:

```
src/
├── Client.php                          # Main client (cURL wrapper)
├── Message.php                         # Message model
├── Request/
│   ├── SendMessagesRequest.php        # Send request
│   ├── CheckRepliesRequest.php        # Check replies request
│   ├── ConfirmRepliesRequest.php      # Confirm replies request
│   ├── CheckDeliveryReportsRequest.php # Delivery reports request
│   └── ConfirmDeliveryReportsRequest.php
├── Response/
│   ├── SendMessagesResponse.php       # Send response
│   ├── CheckRepliesResponse.php       # Replies response
│   ├── Reply.php                       # Reply object
│   └── DeliveryReport.php              # Delivery report object
├── Exceptions/
│   ├── MessageMediaException.php       # Base exception
│   ├── ValidationException.php         # Validation errors
│   ├── AuthenticationException.php     # Auth errors
│   └── ApiException.php                # API errors
└── ServiceProvider.php                 # Laravel service provider
```

---

## Authentication

### Basic Authentication

MessageMedia API uses HTTP Basic Authentication:

```php
// Credentials
$apiKey = 'YOUR_API_KEY';
$apiSecret = 'YOUR_API_SECRET';

// Create Authorization header
$credentials = base64_encode("{$apiKey}:{$apiSecret}");
$authHeader = "Authorization: Basic {$credentials}";
```

### HMAC Authentication (Optional)

For enhanced security, HMAC-SHA256 signatures can be used:

```php
// In cURL wrapper
if (config('messagemedia.use_hmac')) {
    $signature = hash_hmac('sha256', $body, $apiSecret);
    $headers[] = "X-MessageMedia-Signature: {$signature}";
}
```

---

## Error Handling

### Exception Hierarchy

```
Exception
├── MessageMediaException
│   ├── ValidationException (400, 422)
│   ├── AuthenticationException (401, 403)
│   ├── NotFoundException (404)
│   └── ApiException (5xx, network errors)
```

### Error Handling Examples

```php
use Infoxchange\MessageMedia\Exceptions\ValidationException;
use Infoxchange\MessageMedia\Exceptions\AuthenticationException;
use Infoxchange\MessageMedia\Exceptions\ApiException;

try {
    $response = $client->sendMessages($request);
} catch (ValidationException $e) {
    // Handle validation errors (400, 422)
    foreach ($e->errors as $error) {
        echo "Field: " . $error['field'] . " - " . $error['message'];
    }
} catch (AuthenticationException $e) {
    // Handle auth errors (401, 403)
    echo "Authentication failed: " . $e->getMessage();
} catch (ApiException $e) {
    // Handle server errors (5xx)
    echo "API Error: " . $e->getCode();
} catch (Exception $e) {
    // Handle unexpected errors
    echo "Unexpected error: " . $e->getMessage();
}
```

### API Error Response Format

```json
{
  "errors": [
    {
      "message": "Invalid phone number",
      "code": "INVALID_PHONE_NUMBER"
    }
  ]
}
```

---

## Testing Migration

### Unit Tests

Create tests to verify functionality:

```php
namespace Tests\Unit;

use Infoxchange\MessageMedia\Client;
use Infoxchange\MessageMedia\Message;
use Infoxchange\MessageMedia\Request\SendMessagesRequest;
use PHPUnit\Framework\TestCase;

class MessageMediaClientTest extends TestCase
{
    private $client;

    protected function setUp(): void
    {
        $this->client = new Client(
            apiKey: 'test_key',
            apiSecret: 'test_secret'
        );
    }

    public function test_send_message_successfully()
    {
        $request = new SendMessagesRequest();
        $message = new Message();
        $message->content = 'Test';
        $message->destinationNumber = '+61491570156';
        $request->messages = [$message];

        $response = $this->client->sendMessages($request);

        $this->assertNotEmpty($response->messages);
        $this->assertNotEmpty($response->messages[0]->messageId);
    }

    public function test_validation_error_on_missing_content()
    {
        $request = new SendMessagesRequest();
        $message = new Message();
        $message->destinationNumber = '+61491570156';
        // Missing content
        $request->messages = [$message];

        $this->expectException(ValidationException::class);
        $this->client->sendMessages($request);
    }
}
```

### Integration Tests

```bash
# Test against staging API
php artisan test --env=staging

# Test with mock responses
php artisan test --env=testing
```

---

## Package Installation

### Installation Steps

1. **Install via Composer**
   ```bash
   composer require infoxchange/messagemedia-laravel
   ```

2. **Publish Configuration**
   ```bash
   php artisan vendor:publish --provider="Infoxchange\MessageMedia\ServiceProvider"
   ```

3. **Set Environment Variables**
   ```env
   MESSAGEMEDIA_API_KEY=your_key
   MESSAGEMEDIA_API_SECRET=your_secret
   ```

4. **Update Your Code**
   - Replace SDK imports with new package imports
   - Remove old SDK dependency
   - Update client initialization

5. **Run Tests**
   ```bash
   php artisan test
   ```

### Dependency Removal

**Remove from `composer.json`:**
```json
{
  "require": {
    "messagemedia/messages-sdk": "*"  // ← REMOVE THIS
  }
}
```

**Run:**
```bash
composer remove messagemedia/messages-sdk
```

---

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| 401 Unauthorized | Check API key/secret credentials |
| 422 Validation Error | Validate message content and phone numbers |
| Connection Timeout | Check network connectivity and API base URL |
| SSL Certificate Error | Set `MESSAGEMEDIA_VERIFY_SSL=false` (development only) |

### Debug Mode

Enable logging:

```env
MESSAGEMEDIA_DEBUG=true
LOG_CHANNEL=messagemedia
```

This logs all API requests/responses to `storage/logs/messagemedia.log`.

---

## Performance Considerations

### cURL Optimization

- **Connection Pooling**: Reuse client instance for multiple requests
- **Timeout**: Set reasonable timeout (default: 30 seconds)
- **Keep-Alive**: Enabled by default for persistent connections

### Batch Operations

For sending multiple messages efficiently:

```php
$request = new SendMessagesRequest();

// Build up to 100 messages per request
for ($i = 0; $i < 100; $i++) {
    $message = new Message();
    $message->content = "Message $i";
    $message->destinationNumber = "+614915701{$i}";
    $request->messages[] = $message;
}

$response = $client->sendMessages($request);
```

---

## Migration Checklist

- [ ] Environment variables configured
- [ ] Package installed via Composer
- [ ] Service provider registered
- [ ] Old SDK imports updated
- [ ] Client initialization refactored
- [ ] Request/response classes updated
- [ ] Error handling implemented
- [ ] Unit tests passing
- [ ] Integration tests passing
- [ ] Staging environment tested
- [ ] Production deployment ready
- [ ] Old SDK dependency removed
- [ ] Documentation updated
- [ ] Team trained on new package

---

## Support & Resources

- **API Documentation**: https://messagemedia.github.io/documentation/#tag/Messages
- **Support Portal**: https://support.messagemedia.com
- **Package Repository**: https://github.com/infoxchange/messagemedia-laravel
- **Issues/Bugs**: GitHub Issues tracker

---

## License

This package is open-source software licensed under the Apache 2.0 license.
