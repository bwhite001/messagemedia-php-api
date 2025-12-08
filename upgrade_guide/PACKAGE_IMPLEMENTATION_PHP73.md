# MessageMedia Laravel Package - PHP 7.3+ & Laravel 6 Compatible Implementation

**Document Version:** 1.0  
**PHP Compatibility:** >= 7.3.25  
**Laravel Compatibility:** ~6.20.27  
**Status:** Production Ready  

---

## Table of Contents

1. [Key Compatibility Features](#key-compatibility-features)
2. [Complete Source Code](#complete-source-code)
3. [Installation & Setup](#installation--setup)
4. [Migration Guide for Laravel 6](#migration-guide-for-laravel-6)
5. [Testing on PHP 7.3](#testing-on-php-73)
6. [Troubleshooting](#troubleshooting)

---

## Key Compatibility Features

### No PHP 8 Features Used

✅ **No typed properties** - Using `@var` docblocks instead  
✅ **No named arguments** - Traditional positional arguments only  
✅ **No match expressions** - Using switch statements  
✅ **No constructor promotion** - Manual property initialization  
✅ **Traditional type hints** - Compatible with PHP 7.3+  
✅ **No arrow functions** - Using traditional closure syntax  
✅ **No nullsafe operator** - Explicit null checks  

### Laravel 6 Compatibility

✅ **Traditional service provider syntax**  
✅ **Compatible with package discovery**  
✅ **Uses config() helper (available in Laravel 6)**  
✅ **Compatible with Laravel facades**  
✅ **No Laravel 7+ syntax features**  

---

## Complete Source Code

All source files are provided with full PHP 7.3 compatibility. Copy each file to the appropriate directory.

### File 1: config/messagemedia.php

```php
<?php

return [
    /*
     * MessageMedia API credentials
     */
    'api_key' => env('MESSAGEMEDIA_API_KEY'),
    'api_secret' => env('MESSAGEMEDIA_API_SECRET'),

    /*
     * Use HMAC signature authentication (optional)
     */
    'use_hmac' => env('MESSAGEMEDIA_USE_HMAC', false),

    /*
     * MessageMedia API base URL
     */
    'base_url' => env('MESSAGEMEDIA_BASE_URL', 'https://api.messagemedia.com/v1'),

    /*
     * HTTP timeout in seconds
     */
    'timeout' => env('MESSAGEMEDIA_TIMEOUT', 30),

    /*
     * Verify SSL certificates
     */
    'verify_ssl' => env('MESSAGEMEDIA_VERIFY_SSL', true),

    /*
     * Enable debug logging
     */
    'debug' => env('MESSAGEMEDIA_DEBUG', false),
];
```

### File 2: src/Client.php

```php
<?php

namespace Infoxchange\MessageMedia;

use Infoxchange\MessageMedia\Http\HttpClient;
use Infoxchange\MessageMedia\Request\SendMessagesRequest;
use Infoxchange\MessageMedia\Request\CheckRepliesRequest;
use Infoxchange\MessageMedia\Request\CheckDeliveryReportsRequest;
use Infoxchange\MessageMedia\Request\ConfirmRepliesRequest;
use Infoxchange\MessageMedia\Request\ConfirmDeliveryReportsRequest;
use Infoxchange\MessageMedia\Response\SendMessagesResponse;
use Infoxchange\MessageMedia\Response\CheckRepliesResponse;
use Infoxchange\MessageMedia\Exceptions\ValidationException;

class Client
{
    /** @var HttpClient */
    private $httpClient;

    /** @var string */
    private $apiKey;

    /** @var string */
    private $apiSecret;

    /** @var string */
    private $baseUrl;

    /** @var bool */
    private $useHmac;

    /**
     * @param string $apiKey
     * @param string $apiSecret
     * @param string $baseUrl
     * @param bool $useHmac
     */
    public function __construct(
        $apiKey,
        $apiSecret,
        $baseUrl = 'https://api.messagemedia.com/v1',
        $useHmac = false
    ) {
        $this->apiKey = $apiKey;
        $this->apiSecret = $apiSecret;
        $this->baseUrl = rtrim($baseUrl, '/');
        $this->useHmac = $useHmac;

        $this->httpClient = new HttpClient(
            $apiKey,
            $apiSecret,
            $this->baseUrl,
            $useHmac
        );
    }

    /**
     * Send messages via MessageMedia API
     *
     * @param SendMessagesRequest $request
     * @return SendMessagesResponse
     * @throws ValidationException
     * @throws \Infoxchange\MessageMedia\Exceptions\AuthenticationException
     * @throws \Infoxchange\MessageMedia\Exceptions\ApiException
     */
    public function sendMessages(SendMessagesRequest $request)
    {
        $this->validateSendMessagesRequest($request);

        $messages = array_map(function ($msg) {
            return $this->messageToArray($msg);
        }, $request->messages);

        $payload = json_encode(['messages' => $messages]);

        $response = $this->httpClient->post('/messages/send', $payload);

        return SendMessagesResponse::fromArray($response);
    }

    /**
     * Check for incoming replies
     *
     * @param CheckRepliesRequest $request
     * @return CheckRepliesResponse
     */
    public function checkReplies(CheckRepliesRequest $request)
    {
        $query = [];

        if ($request->limit !== null) {
            $query['limit'] = $request->limit;
        }
        if ($request->offset !== null) {
            $query['offset'] = $request->offset;
        }

        $response = $this->httpClient->get('/replies', $query);

        return CheckRepliesResponse::fromArray($response);
    }

    /**
     * Confirm replies have been processed
     *
     * @param ConfirmRepliesRequest $request
     * @return void
     */
    public function confirmReplies(ConfirmRepliesRequest $request)
    {
        $payload = json_encode(['reply_ids' => $request->replyIds]);
        $this->httpClient->put('/replies', $payload);
    }

    /**
     * Check delivery reports for sent messages
     *
     * @param CheckDeliveryReportsRequest $request
     * @return \Infoxchange\MessageMedia\Response\CheckDeliveryReportsResponse
     */
    public function checkDeliveryReports(CheckDeliveryReportsRequest $request)
    {
        $query = [];

        if ($request->limit !== null) {
            $query['limit'] = $request->limit;
        }
        if ($request->offset !== null) {
            $query['offset'] = $request->offset;
        }

        $response = $this->httpClient->get('/delivery-reports', $query);

        return \Infoxchange\MessageMedia\Response\CheckDeliveryReportsResponse::fromArray($response);
    }

    /**
     * Confirm delivery reports have been processed
     *
     * @param ConfirmDeliveryReportsRequest $request
     * @return void
     */
    public function confirmDeliveryReports(ConfirmDeliveryReportsRequest $request)
    {
        $payload = json_encode(['delivery_report_ids' => $request->deliveryReportIds]);
        $this->httpClient->put('/delivery-reports', $payload);
    }

    /**
     * Validate SendMessagesRequest
     *
     * @param SendMessagesRequest $request
     * @throws ValidationException
     */
    private function validateSendMessagesRequest(SendMessagesRequest $request)
    {
        if (empty($request->messages)) {
            throw new ValidationException([
                ['field' => 'messages', 'message' => 'At least one message is required'],
            ]);
        }

        $errors = [];

        foreach ($request->messages as $index => $message) {
            if (empty($message->content)) {
                $errors[] = [
                    'field' => "messages.{$index}.content",
                    'message' => 'Message content is required',
                ];
            }

            if (empty($message->destinationNumber)) {
                $errors[] = [
                    'field' => "messages.{$index}.destinationNumber",
                    'message' => 'Destination number is required',
                ];
            }

            $destNumber = $message->destinationNumber ?? '';
            if (!preg_match('/^\+?[0-9]{10,}$/', $destNumber)) {
                $errors[] = [
                    'field' => "messages.{$index}.destinationNumber",
                    'message' => 'Invalid phone number format',
                ];
            }
        }

        if (!empty($errors)) {
            throw new ValidationException($errors);
        }
    }

    /**
     * Convert Message object to array
     *
     * @param Message $message
     * @return array
     */
    private function messageToArray(Message $message)
    {
        $data = [
            'content' => $message->content,
            'destination_number' => $message->destinationNumber,
        ];

        if (!empty($message->sourceNumber)) {
            $data['source_number'] = $message->sourceNumber;
        }

        if (!empty($message->callbackUrl)) {
            $data['callback_url'] = $message->callbackUrl;
        }

        if (!empty($message->scheduledDatetime)) {
            $data['scheduled_datetime'] = $message->scheduledDatetime;
        }

        if (!empty($message->metadata)) {
            $data['metadata'] = $message->metadata;
        }

        if (!empty($message->deliveryReportUrl)) {
            $data['delivery_report_url'] = $message->deliveryReportUrl;
        }

        if (!empty($message->messageExpiryTimestamp)) {
            $data['message_expiry_timestamp'] = $message->messageExpiryTimestamp;
        }

        return $data;
    }
}
```

### File 3: src/Message.php

```php
<?php

namespace Infoxchange\MessageMedia;

class Message
{
    /** @var string|null */
    public $content;

    /** @var string|null */
    public $destinationNumber;

    /** @var string|null */
    public $sourceNumber;

    /** @var string|null */
    public $callbackUrl;

    /** @var string|null */
    public $scheduledDatetime;

    /** @var array|null */
    public $metadata;

    /** @var string|null */
    public $deliveryReportUrl;

    /** @var int|null */
    public $messageExpiryTimestamp;

    /** @var string|null */
    public $messageId;

    /**
     * @param array|null $attributes
     */
    public function __construct($attributes = null)
    {
        if ($attributes) {
            foreach ($attributes as $key => $value) {
                if (property_exists($this, $key)) {
                    $this->$key = $value;
                }
            }
        }
    }

    /**
     * @param array $data
     * @return self
     */
    public static function fromArray(array $data)
    {
        $message = new self();

        $message->content = $data['content'] ?? null;
        $message->destinationNumber = $data['destination_number'] ?? null;
        $message->sourceNumber = $data['source_number'] ?? null;
        $message->callbackUrl = $data['callback_url'] ?? null;
        $message->scheduledDatetime = $data['scheduled_datetime'] ?? null;
        $message->metadata = $data['metadata'] ?? null;
        $message->deliveryReportUrl = $data['delivery_report_url'] ?? null;
        $message->messageExpiryTimestamp = $data['message_expiry_timestamp'] ?? null;
        $message->messageId = $data['message_id'] ?? null;

        return $message;
    }

    /**
     * @return array
     */
    public function toArray()
    {
        return [
            'content' => $this->content,
            'destination_number' => $this->destinationNumber,
            'source_number' => $this->sourceNumber,
            'callback_url' => $this->callbackUrl,
            'scheduled_datetime' => $this->scheduledDatetime,
            'metadata' => $this->metadata,
            'delivery_report_url' => $this->deliveryReportUrl,
            'message_expiry_timestamp' => $this->messageExpiryTimestamp,
            'message_id' => $this->messageId,
        ];
    }
}
```

### File 4: src/Http/HttpClient.php

```php
<?php

namespace Infoxchange\MessageMedia\Http;

use Infoxchange\MessageMedia\Exceptions\AuthenticationException;
use Infoxchange\MessageMedia\Exceptions\NotFoundException;
use Infoxchange\MessageMedia\Exceptions\ValidationException;
use Infoxchange\MessageMedia\Exceptions\ApiException;

class HttpClient
{
    /** @var string */
    private $apiKey;

    /** @var string */
    private $apiSecret;

    /** @var string */
    private $baseUrl;

    /** @var bool */
    private $useHmac;

    /** @var int */
    private $timeout;

    /** @var bool */
    private $verifySsl;

    /**
     * @param string $apiKey
     * @param string $apiSecret
     * @param string $baseUrl
     * @param bool $useHmac
     * @param int $timeout
     * @param bool $verifySsl
     */
    public function __construct(
        $apiKey,
        $apiSecret,
        $baseUrl = 'https://api.messagemedia.com/v1',
        $useHmac = false,
        $timeout = 30,
        $verifySsl = true
    ) {
        $this->apiKey = $apiKey;
        $this->apiSecret = $apiSecret;
        $this->baseUrl = rtrim($baseUrl, '/');
        $this->useHmac = $useHmac;
        $this->timeout = $timeout;
        $this->verifySsl = $verifySsl;
    }

    /**
     * @param string $endpoint
     * @param array $query
     * @return array
     */
    public function get($endpoint, array $query = [])
    {
        $url = $this->buildUrl($endpoint, $query);
        return $this->request('GET', $url, null);
    }

    /**
     * @param string $endpoint
     * @param string|null $body
     * @return array
     */
    public function post($endpoint, $body = null)
    {
        $url = $this->buildUrl($endpoint);
        return $this->request('POST', $url, $body);
    }

    /**
     * @param string $endpoint
     * @param string|null $body
     * @return array
     */
    public function put($endpoint, $body = null)
    {
        $url = $this->buildUrl($endpoint);
        return $this->request('PUT', $url, $body);
    }

    /**
     * @param string $endpoint
     * @return array
     */
    public function delete($endpoint)
    {
        $url = $this->buildUrl($endpoint);
        return $this->request('DELETE', $url, null);
    }

    /**
     * @param string $method
     * @param string $url
     * @param string|null $body
     * @return array
     * @throws AuthenticationException
     * @throws ValidationException
     * @throws NotFoundException
     * @throws ApiException
     */
    private function request($method, $url, $body = null)
    {
        $curl = curl_init();

        curl_setopt($curl, CURLOPT_CUSTOMREQUEST, $method);
        curl_setopt($curl, CURLOPT_URL, $url);

        $headers = $this->buildHeaders($body);
        curl_setopt($curl, CURLOPT_HTTPHEADER, $headers);
        curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($curl, CURLOPT_TIMEOUT, $this->timeout);
        curl_setopt($curl, CURLOPT_CONNECTTIMEOUT, 10);

        if (!$this->verifySsl) {
            curl_setopt($curl, CURLOPT_SSL_VERIFYPEER, false);
            curl_setopt($curl, CURLOPT_SSL_VERIFYHOST, false);
        }

        if ($body !== null && in_array($method, ['POST', 'PUT'])) {
            curl_setopt($curl, CURLOPT_POSTFIELDS, $body);
        }

        $response = curl_exec($curl);
        $httpCode = curl_getinfo($curl, CURLINFO_HTTP_CODE);
        $curlError = curl_error($curl);

        curl_close($curl);

        if ($response === false) {
            throw new ApiException("cURL Error: {$curlError}");
        }

        $data = json_decode($response, true);

        $this->handleHttpError($httpCode, $data);

        return $data ?? [];
    }

    /**
     * @param string $endpoint
     * @param array $query
     * @return string
     */
    private function buildUrl($endpoint, array $query = [])
    {
        $url = $this->baseUrl . '/' . ltrim($endpoint, '/');

        if (!empty($query)) {
            $url .= '?' . http_build_query($query);
        }

        return $url;
    }

    /**
     * @param string|null $body
     * @return array
     */
    private function buildHeaders($body = null)
    {
        $headers = [
            'Accept: application/json',
            'Content-Type: application/json',
        ];

        $credentials = base64_encode("{$this->apiKey}:{$this->apiSecret}");
        $headers[] = "Authorization: Basic {$credentials}";

        if ($this->useHmac && $body) {
            $signature = hash_hmac('sha256', $body, $this->apiSecret);
            $headers[] = "X-MessageMedia-Signature: {$signature}";
        }

        return $headers;
    }

    /**
     * @param int $httpCode
     * @param array|null $data
     * @throws AuthenticationException
     * @throws ValidationException
     * @throws NotFoundException
     * @throws ApiException
     */
    private function handleHttpError($httpCode, $data = null)
    {
        if ($httpCode < 400) {
            return;
        }

        $message = isset($data['message']) ? $data['message'] : 'Unknown error';
        $errors = isset($data['errors']) ? $data['errors'] : [];

        switch ($httpCode) {
            case 401:
            case 403:
                throw new AuthenticationException($message);
            case 404:
                throw new NotFoundException($message);
            case 400:
            case 422:
                throw new ValidationException($errors);
            default:
                throw new ApiException($message, $httpCode);
        }
    }
}
```

### File 5: src/Request Classes

Create five separate files in `src/Request/`:

**SendMessagesRequest.php:**
```php
<?php

namespace Infoxchange\MessageMedia\Request;

class SendMessagesRequest
{
    /** @var array */
    public $messages = [];

    /**
     * @param array|null $messages
     */
    public function __construct($messages = null)
    {
        if ($messages) {
            $this->messages = $messages;
        }
    }
}
```

**CheckRepliesRequest.php:**
```php
<?php

namespace Infoxchange\MessageMedia\Request;

class CheckRepliesRequest
{
    /** @var int|null */
    public $limit;

    /** @var int|null */
    public $offset;
}
```

**ConfirmRepliesRequest.php:**
```php
<?php

namespace Infoxchange\MessageMedia\Request;

class ConfirmRepliesRequest
{
    /** @var array */
    public $replyIds = [];

    /**
     * @param array $replyIds
     */
    public function __construct(array $replyIds = [])
    {
        $this->replyIds = $replyIds;
    }
}
```

**CheckDeliveryReportsRequest.php:**
```php
<?php

namespace Infoxchange\MessageMedia\Request;

class CheckDeliveryReportsRequest
{
    /** @var int|null */
    public $limit;

    /** @var int|null */
    public $offset;
}
```

**ConfirmDeliveryReportsRequest.php:**
```php
<?php

namespace Infoxchange\MessageMedia\Request;

class ConfirmDeliveryReportsRequest
{
    /** @var array */
    public $deliveryReportIds = [];

    /**
     * @param array $deliveryReportIds
     */
    public function __construct(array $deliveryReportIds = [])
    {
        $this->deliveryReportIds = $deliveryReportIds;
    }
}
```

### File 6: src/Response Classes

Create files in `src/Response/`:

**SendMessagesResponse.php:**
```php
<?php

namespace Infoxchange\MessageMedia\Response;

use Infoxchange\MessageMedia\Message;

class SendMessagesResponse
{
    /** @var array */
    public $messages = [];

    /**
     * @param array $data
     * @return self
     */
    public static function fromArray(array $data)
    {
        $response = new self();

        if (isset($data['messages'])) {
            foreach ($data['messages'] as $msgData) {
                $response->messages[] = Message::fromArray($msgData);
            }
        }

        return $response;
    }
}
```

**CheckRepliesResponse.php:**
```php
<?php

namespace Infoxchange\MessageMedia\Response;

class CheckRepliesResponse
{
    /** @var array */
    public $replies = [];

    /** @var int|null */
    public $pageSize;

    /** @var int|null */
    public $pageNumber;

    /**
     * @param array $data
     * @return self
     */
    public static function fromArray(array $data)
    {
        $response = new self();

        if (isset($data['replies'])) {
            foreach ($data['replies'] as $replyData) {
                $response->replies[] = Reply::fromArray($replyData);
            }
        }

        $response->pageSize = $data['page_size'] ?? null;
        $response->pageNumber = $data['page_number'] ?? null;

        return $response;
    }
}
```

**Reply.php:**
```php
<?php

namespace Infoxchange\MessageMedia\Response;

class Reply
{
    /** @var string|null */
    public $replyId;

    /** @var string|null */
    public $messageId;

    /** @var string|null */
    public $content;

    /** @var string|null */
    public $sourceNumber;

    /** @var string|null */
    public $destinationNumber;

    /** @var string|null */
    public $dateReceived;

    /** @var array|null */
    public $metadata;

    /**
     * @param array $data
     * @return self
     */
    public static function fromArray(array $data)
    {
        $reply = new self();

        $reply->replyId = $data['reply_id'] ?? null;
        $reply->messageId = $data['message_id'] ?? null;
        $reply->content = $data['content'] ?? null;
        $reply->sourceNumber = $data['source_number'] ?? null;
        $reply->destinationNumber = $data['destination_number'] ?? null;
        $reply->dateReceived = $data['date_received'] ?? null;
        $reply->metadata = $data['metadata'] ?? null;

        return $reply;
    }
}
```

**DeliveryReport.php:**
```php
<?php

namespace Infoxchange\MessageMedia\Response;

class DeliveryReport
{
    /** @var string|null */
    public $deliveryReportId;

    /** @var string|null */
    public $messageId;

    /** @var string|null */
    public $status;

    /** @var string|null */
    public $statusCode;

    /** @var string|null */
    public $dateReceived;

    /** @var string|null */
    public $sourceNumber;

    /** @var string|null */
    public $destinationNumber;

    /** @var array|null */
    public $metadata;

    /**
     * @param array $data
     * @return self
     */
    public static function fromArray(array $data)
    {
        $report = new self();

        $report->deliveryReportId = $data['delivery_report_id'] ?? null;
        $report->messageId = $data['message_id'] ?? null;
        $report->status = $data['status'] ?? null;
        $report->statusCode = $data['status_code'] ?? null;
        $report->dateReceived = $data['date_received'] ?? null;
        $report->sourceNumber = $data['source_number'] ?? null;
        $report->destinationNumber = $data['destination_number'] ?? null;
        $report->metadata = $data['metadata'] ?? null;

        return $report;
    }
}
```

**CheckDeliveryReportsResponse.php:**
```php
<?php

namespace Infoxchange\MessageMedia\Response;

class CheckDeliveryReportsResponse
{
    /** @var array */
    public $deliveryReports = [];

    /** @var int|null */
    public $pageSize;

    /** @var int|null */
    public $pageNumber;

    /**
     * @param array $data
     * @return self
     */
    public static function fromArray(array $data)
    {
        $response = new self();

        if (isset($data['delivery_reports'])) {
            foreach ($data['delivery_reports'] as $reportData) {
                $response->deliveryReports[] = DeliveryReport::fromArray($reportData);
            }
        }

        $response->pageSize = $data['page_size'] ?? null;
        $response->pageNumber = $data['page_number'] ?? null;

        return $response;
    }
}
```

### File 7: src/Exceptions

Create files in `src/Exceptions/`:

**MessageMediaException.php:**
```php
<?php

namespace Infoxchange\MessageMedia\Exceptions;

class MessageMediaException extends \Exception
{
}
```

**ValidationException.php:**
```php
<?php

namespace Infoxchange\MessageMedia\Exceptions;

class ValidationException extends MessageMediaException
{
    /** @var array */
    public $errors = [];

    /**
     * @param array $errors
     * @param string $message
     * @param int $code
     */
    public function __construct(array $errors = [], $message = '', $code = 422)
    {
        $this->errors = $errors;

        if (empty($message)) {
            $message = 'Validation failed: ' . json_encode($errors);
        }

        parent::__construct($message, $code);
    }
}
```

**AuthenticationException.php:**
```php
<?php

namespace Infoxchange\MessageMedia\Exceptions;

class AuthenticationException extends MessageMediaException
{
    /**
     * @param string $message
     * @param int $code
     */
    public function __construct($message = 'Authentication failed', $code = 401)
    {
        parent::__construct($message, $code);
    }
}
```

**NotFoundException.php:**
```php
<?php

namespace Infoxchange\MessageMedia\Exceptions;

class NotFoundException extends MessageMediaException
{
    /**
     * @param string $message
     * @param int $code
     */
    public function __construct($message = 'Resource not found', $code = 404)
    {
        parent::__construct($message, $code);
    }
}
```

**ApiException.php:**
```php
<?php

namespace Infoxchange\MessageMedia\Exceptions;

class ApiException extends MessageMediaException
{
    /**
     * @param string $message
     * @param int $code
     */
    public function __construct($message = 'API error', $code = 500)
    {
        parent::__construct($message, $code);
    }
}
```

### File 8: src/ServiceProvider.php

```php
<?php

namespace Infoxchange\MessageMedia;

use Illuminate\Support\ServiceProvider as LaravelServiceProvider;

class ServiceProvider extends LaravelServiceProvider
{
    /**
     * Register services.
     *
     * @return void
     */
    public function register()
    {
        $this->mergeConfigFrom(
            __DIR__ . '/../config/messagemedia.php',
            'messagemedia'
        );

        $this->app->singleton('messagemedia', function ($app) {
            return new Client(
                config('messagemedia.api_key'),
                config('messagemedia.api_secret'),
                config('messagemedia.base_url'),
                config('messagemedia.use_hmac', false)
            );
        });
    }

    /**
     * Bootstrap services.
     *
     * @return void
     */
    public function boot()
    {
        $this->publishes([
            __DIR__ . '/../config/messagemedia.php' => config_path('messagemedia.php'),
        ], 'config');
    }
}
```

### File 9: src/Facades/MessageMedia.php

```php
<?php

namespace Infoxchange\MessageMedia\Facades;

use Illuminate\Support\Facades\Facade;

class MessageMedia extends Facade
{
    /**
     * Get the registered name of the component.
     *
     * @return string
     */
    protected static function getFacadeAccessor()
    {
        return 'messagemedia';
    }
}
```

---

## Installation & Setup

### Step 1: Create Package Directory

```bash
mkdir -p packages/messagemedia-laravel/src/{Http,Request,Response,Exceptions,Facades}
mkdir -p packages/messagemedia-laravel/config
```

### Step 2: Copy All Files

Copy all files from this document to their respective directories.

### Step 3: Create composer.json

```json
{
  "name": "infoxchange/messagemedia-laravel",
  "description": "Laravel package for MessageMedia Messaging API (PHP 7.3+ compatible)",
  "keywords": ["sms", "messaging", "messagemedia", "sinch", "laravel"],
  "license": "Apache-2.0",
  "authors": [
    {
      "name": "Infoxchange Technology",
      "email": "dev@infoxchange.org"
    }
  ],
  "require": {
    "php": ">=7.3.25",
    "laravel/framework": "~6.20.27",
    "ext-curl": "*",
    "ext-json": "*"
  },
  "autoload": {
    "psr-4": {
      "Infoxchange\\MessageMedia\\": "src/"
    }
  },
  "extra": {
    "laravel": {
      "providers": [
        "Infoxchange\\MessageMedia\\ServiceProvider"
      ],
      "aliases": {
        "MessageMedia": "Infoxchange\\MessageMedia\\Facades\\MessageMedia"
      }
    }
  }
}
```

### Step 4: Register Package in Main Application

Update root `composer.json`:

```json
{
  "repositories": [
    {
      "type": "path",
      "url": "packages/messagemedia-laravel"
    }
  ],
  "require": {
    "infoxchange/messagemedia-laravel": "*"
  }
}
```

### Step 5: Install and Publish

```bash
# Install package
composer update

# Publish configuration
php artisan vendor:publish --provider="Infoxchange\MessageMedia\ServiceProvider"

# Update .env
cat >> .env << EOF
MESSAGEMEDIA_API_KEY=your_api_key_here
MESSAGEMEDIA_API_SECRET=your_api_secret_here
MESSAGEMEDIA_USE_HMAC=false
MESSAGEMEDIA_BASE_URL=https://api.messagemedia.com/v1
EOF
```

---

## Migration Guide for Laravel 6

### From Deprecated SDK to New Package

#### Before (Old SDK):
```php
<?php

use MessageMediaMessagesLib\MessageMediaMessagesClient;
use MessageMediaMessagesLib\Models\SendMessagesRequest;
use MessageMediaMessagesLib\Models\Message;

$client = new MessageMediaMessagesClient('KEY', 'SECRET');
$controller = $client->getMessages();

$body = new SendMessagesRequest();
$body->messages = [];

$message = new Message();
$message->content = 'Hello!';
$message->destinationNumber = '+61491570156';
$body->messages[] = $message;

try {
    $result = $controller->sendMessages($body);
} catch (Exception $e) {
    // Handle error
}
```

#### After (New Package):
```php
<?php

use Infoxchange\MessageMedia\Facades\MessageMedia;
use Infoxchange\MessageMedia\Request\SendMessagesRequest;
use Infoxchange\MessageMedia\Message;

$request = new SendMessagesRequest();

$message = new Message();
$message->content = 'Hello!';
$message->destinationNumber = '+61491570156';
$request->messages = [$message];

try {
    $response = MessageMedia::sendMessages($request);
} catch (Exception $e) {
    // Handle error
}
```

### Refactoring Checklist for Laravel 6

- [ ] Update all SDK imports to new package namespace
- [ ] Replace client initialization
- [ ] Remove controller instantiation (use package directly)
- [ ] Update request/response handling
- [ ] Update error handling
- [ ] Test on PHP 7.3 environment
- [ ] Test in Laravel 6.20 environment

---

## Testing on PHP 7.3

### Verify PHP Version

```bash
php -v
# Should output: PHP 7.3.25 or higher
```

### Check Required Extensions

```bash
php -m | grep curl
php -m | grep json
# Both should be listed
```

### Run Tests

```bash
# For Laravel 6, use:
./vendor/bin/phpunit tests/

# Or with artisan (if available):
php artisan test
```

### Common PHP 7.3 Issues

**Issue: "Parse error: syntax error"**  
**Solution:** Check for PHP 8 syntax (typed properties, named arguments, etc.)

**Issue: "Undefined constant"**  
**Solution:** Ensure all constants are properly defined with `const` keyword

**Issue: "Call to undefined function"**  
**Solution:** Check that function is available in PHP 7.3

---

## Troubleshooting

### Issue: "Class not found" Error

**Solution:**
```bash
# Clear composer cache
composer clear-cache

# Regenerate autoloader
composer dump-autoload

# Verify package is installed
composer show infoxchange/messagemedia-laravel
```

### Issue: "API Authentication Failed"

**Solution:**
```bash
# Verify credentials in .env
cat .env | grep MESSAGEMEDIA

# Test credentials manually
php artisan tinker
>>> config('messagemedia.api_key')
>>> config('messagemedia.api_secret')
```

### Issue: "cURL Error"

**Solution:**
```bash
# Verify cURL is installed
php -m | grep curl

# Test cURL connection
php -r "echo curl_version()['version'];"

# Check network connectivity
php -r "echo gethostbyname('api.messagemedia.com');"
```

---

## Next Steps

1. Copy all source files to your package directory
2. Install and publish configuration
3. Set environment variables
4. Run smoke tests on PHP 7.3
5. Migrate existing code
6. Deploy to production

---

**Document Status:** Production Ready for PHP 7.3+ & Laravel 6  
**Last Updated:** December 2025
