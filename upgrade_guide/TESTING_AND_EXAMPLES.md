# MessageMedia Laravel Package - Quick Reference & Testing Guide

---

## Quick Reference

### Installation & Setup

```bash
# 1. Install package
composer require infoxchange/messagemedia-laravel

# 2. Publish configuration
php artisan vendor:publish --provider="Infoxchange\MessageMedia\ServiceProvider"

# 3. Set environment variables
cat > .env.local << EOF
MESSAGEMEDIA_API_KEY=your_api_key_here
MESSAGEMEDIA_API_SECRET=your_api_secret_here
EOF

# 4. Optional: Enable HMAC
echo "MESSAGEMEDIA_USE_HMAC=false" >> .env
```

### Basic Usage

```php
use Infoxchange\MessageMedia\Facades\MessageMedia;
use Infoxchange\MessageMedia\Request\SendMessagesRequest;
use Infoxchange\MessageMedia\Message;

// Create request
$request = new SendMessagesRequest();

// Create message
$message = new Message();
$message->content = 'Hello!';
$message->destinationNumber = '+61491570156';
$request->messages = [$message];

// Send
try {
    $response = MessageMedia::sendMessages($request);
} catch (Exception $e) {
    // Handle error
}
```

---

## Testing Guide

### Unit Tests for Client

```php
// tests/Unit/ClientTest.php

namespace Tests\Unit;

use Infoxchange\MessageMedia\Client;
use Infoxchange\MessageMedia\Message;
use Infoxchange\MessageMedia\Request\SendMessagesRequest;
use Infoxchange\MessageMedia\Exceptions\ValidationException;
use PHPUnit\Framework\TestCase;

class ClientTest extends TestCase
{
    private Client $client;

    protected function setUp(): void
    {
        $this->client = new Client(
            apiKey: 'test_key',
            apiSecret: 'test_secret',
            baseUrl: 'https://api.messagemedia.com/v1'
        );
    }

    /**
     * Test validation: missing content
     */
    public function test_send_messages_requires_content(): void
    {
        $this->expectException(ValidationException::class);

        $request = new SendMessagesRequest();
        $message = new Message();
        $message->destinationNumber = '+61491570156';
        // Missing content
        $request->messages = [$message];

        $this->client->sendMessages($request);
    }

    /**
     * Test validation: missing destination number
     */
    public function test_send_messages_requires_destination(): void
    {
        $this->expectException(ValidationException::class);

        $request = new SendMessagesRequest();
        $message = new Message();
        $message->content = 'Test message';
        // Missing destination
        $request->messages = [$message];

        $this->client->sendMessages($request);
    }

    /**
     * Test validation: invalid phone number
     */
    public function test_send_messages_validates_phone_number(): void
    {
        $this->expectException(ValidationException::class);

        $request = new SendMessagesRequest();
        $message = new Message();
        $message->content = 'Test message';
        $message->destinationNumber = 'invalid-number';
        $request->messages = [$message];

        $this->client->sendMessages($request);
    }

    /**
     * Test validation: empty messages
     */
    public function test_send_messages_requires_messages(): void
    {
        $this->expectException(ValidationException::class);

        $request = new SendMessagesRequest();
        $request->messages = [];

        $this->client->sendMessages($request);
    }

    /**
     * Test message conversion to array
     */
    public function test_message_to_array(): void
    {
        $message = new Message();
        $message->content = 'Test';
        $message->destinationNumber = '+61491570156';
        $message->sourceNumber = '+61491570157';
        $message->metadata = ['key' => 'value'];

        $array = $message->toArray();

        $this->assertEquals('Test', $array['content']);
        $this->assertEquals('+61491570156', $array['destination_number']);
        $this->assertEquals('+61491570157', $array['source_number']);
        $this->assertEquals(['key' => 'value'], $array['metadata']);
    }
}
```

### Feature Tests (Mock HTTP)

```php
// tests/Feature/SendMessagesTest.php

namespace Tests\Feature;

use Infoxchange\MessageMedia\Client;
use Infoxchange\MessageMedia\Message;
use Infoxchange\MessageMedia\Request\SendMessagesRequest;
use PHPUnit\Framework\TestCase;

class SendMessagesTest extends TestCase
{
    /**
     * Mock HTTP client for testing
     */
    public function test_send_messages_with_mock_response(): void
    {
        // Mock cURL response
        $mockResponse = [
            'messages' => [
                [
                    'message_id' => 'd15b87c0-1fb5-11e8-a871-00007f124ef0',
                    'status' => 'queued',
                    'destination_number' => '+61491570156',
                ]
            ]
        ];

        // This would typically use a mock HTTP client
        // For now, test the response parsing

        $response = \Infoxchange\MessageMedia\Response\SendMessagesResponse::fromArray($mockResponse);

        $this->assertCount(1, $response->messages);
        $this->assertEquals('d15b87c0-1fb5-11e8-a871-00007f124ef0', $response->messages[0]->messageId);
    }
}
```

### Integration Tests

```php
// tests/Integration/MessageMediaIntegrationTest.php

namespace Tests\Integration;

use Infoxchange\MessageMedia\Client;
use Infoxchange\MessageMedia\Message;
use Infoxchange\MessageMedia\Request\SendMessagesRequest;
use Infoxchange\MessageMedia\Exceptions\AuthenticationException;
use PHPUnit\Framework\TestCase;

class MessageMediaIntegrationTest extends TestCase
{
    private Client $client;

    protected function setUp(): void
    {
        // Use test credentials
        $this->client = new Client(
            apiKey: getenv('MESSAGEMEDIA_API_KEY') ?: 'test_key',
            apiSecret: getenv('MESSAGEMEDIA_API_SECRET') ?: 'test_secret',
        );
    }

    /**
     * Test actual API call (requires valid credentials)
     * 
     * @group integration
     */
    public function test_send_message_to_api(): void
    {
        if (!getenv('MESSAGEMEDIA_API_KEY')) {
            $this->markTestSkipped('MessageMedia credentials not configured');
        }

        $request = new SendMessagesRequest();
        $message = new Message();
        $message->content = 'Test from integration suite';
        $message->destinationNumber = '+61491570156';
        $request->messages = [$message];

        try {
            $response = $this->client->sendMessages($request);
            
            $this->assertNotEmpty($response->messages);
            $this->assertNotEmpty($response->messages[0]->messageId);
        } catch (AuthenticationException $e) {
            $this->markTestSkipped('Invalid API credentials: ' . $e->getMessage());
        }
    }
}
```

---

## Laravel Service Class Example

```php
// app/Services/SmsService.php

namespace App\Services;

use Infoxchange\MessageMedia\Client;
use Infoxchange\MessageMedia\Message;
use Infoxchange\MessageMedia\Request\SendMessagesRequest;
use Infoxchange\MessageMedia\Exceptions\ValidationException;
use Infoxchange\MessageMedia\Exceptions\AuthenticationException;
use Illuminate\Support\Facades\Log;

class SmsService
{
    public function __construct(private Client $messagemedia)
    {
    }

    /**
     * Send SMS to single recipient
     */
    public function sendSms(string $phoneNumber, string $message, array $metadata = []): string
    {
        $request = new SendMessagesRequest();
        
        $sms = new Message();
        $sms->content = $message;
        $sms->destinationNumber = $phoneNumber;
        $sms->metadata = $metadata;
        
        $request->messages = [$sms];

        try {
            $response = $this->messagemedia->sendMessages($request);
            
            Log::info('SMS sent', [
                'message_id' => $response->messages[0]->messageId,
                'destination' => $phoneNumber,
            ]);

            return $response->messages[0]->messageId;
        } catch (ValidationException $e) {
            Log::error('SMS validation failed', ['errors' => $e->errors]);
            throw new \InvalidArgumentException('Invalid SMS parameters: ' . $e->getMessage());
        } catch (AuthenticationException $e) {
            Log::error('SMS authentication failed', ['error' => $e->getMessage()]);
            throw new \RuntimeException('MessageMedia authentication failed');
        }
    }

    /**
     * Send SMS to multiple recipients
     */
    public function sendBulkSms(array $recipients, string $message): array
    {
        $request = new SendMessagesRequest();

        foreach ($recipients as $phoneNumber) {
            $sms = new Message();
            $sms->content = $message;
            $sms->destinationNumber = $phoneNumber;
            $request->messages[] = $sms;
        }

        $response = $this->messagemedia->sendMessages($request);

        return array_map(
            fn($msg) => ['message_id' => $msg->messageId, 'destination' => $msg->destinationNumber],
            $response->messages
        );
    }

    /**
     * Send SMS with callback URL
     */
    public function sendSmsWithCallback(
        string $phoneNumber,
        string $message,
        string $callbackUrl,
        array $metadata = []
    ): string {
        $request = new SendMessagesRequest();
        
        $sms = new Message();
        $sms->content = $message;
        $sms->destinationNumber = $phoneNumber;
        $sms->callbackUrl = $callbackUrl;
        $sms->metadata = $metadata;
        
        $request->messages = [$sms];

        $response = $this->messagemedia->sendMessages($request);
        
        return $response->messages[0]->messageId;
    }

    /**
     * Check for incoming replies
     */
    public function checkReplies(int $limit = 10): array
    {
        $request = new \Infoxchange\MessageMedia\Request\CheckRepliesRequest();
        $request->limit = $limit;

        $response = $this->messagemedia->checkReplies($request);

        return $response->replies;
    }

    /**
     * Get delivery reports
     */
    public function getDeliveryReports(int $limit = 10): array
    {
        $request = new \Infoxchange\MessageMedia\Request\CheckDeliveryReportsRequest();
        $request->limit = $limit;

        $response = $this->messagemedia->checkDeliveryReports($request);

        return $response->deliveryReports;
    }
}
```

### Using the Service in a Controller

```php
// app/Http/Controllers/NotificationController.php

namespace App\Http\Controllers;

use App\Services\SmsService;
use Illuminate\Http\Request;

class NotificationController extends Controller
{
    public function __construct(private SmsService $smsService)
    {
    }

    /**
     * Send notification SMS
     */
    public function sendNotification(Request $request)
    {
        $validated = $request->validate([
            'phone_number' => 'required|regex:/^\+?[0-9]{10,}$/',
            'message' => 'required|string|max:160',
        ]);

        try {
            $messageId = $this->smsService->sendSms(
                $validated['phone_number'],
                $validated['message'],
                ['request_id' => $request->id()]
            );

            return response()->json([
                'success' => true,
                'message_id' => $messageId,
            ]);
        } catch (\Exception $e) {
            return response()->json([
                'success' => false,
                'error' => $e->getMessage(),
            ], 400);
        }
    }

    /**
     * Check SMS replies
     */
    public function checkReplies()
    {
        $replies = $this->smsService->checkReplies(limit: 20);

        return response()->json([
            'count' => count($replies),
            'replies' => $replies,
        ]);
    }

    /**
     * Get delivery status
     */
    public function getDeliveryStatus()
    {
        $reports = $this->smsService->getDeliveryReports(limit: 50);

        return response()->json([
            'count' => count($reports),
            'reports' => $reports,
        ]);
    }
}
```

### Webhook Handler for Delivery Notifications

```php
// app/Http/Controllers/WebhookController.php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;

class WebhookController extends Controller
{
    /**
     * Handle MessageMedia delivery report webhook
     */
    public function handleDeliveryReport(Request $request)
    {
        $data = $request->json()->all();

        Log::info('MessageMedia delivery report received', $data);

        // Process delivery reports
        if (isset($data['delivery_reports'])) {
            foreach ($data['delivery_reports'] as $report) {
                $this->updateMessageStatus(
                    $report['message_id'],
                    $report['status'],
                    $report['status_code'] ?? null
                );
            }
        }

        return response()->json(['success' => true]);
    }

    /**
     * Handle MessageMedia reply webhook
     */
    public function handleReply(Request $request)
    {
        $data = $request->json()->all();

        Log::info('MessageMedia reply received', $data);

        // Process incoming replies
        if (isset($data['replies'])) {
            foreach ($data['replies'] as $reply) {
                $this->storeIncomingReply(
                    $reply['reply_id'],
                    $reply['message_id'],
                    $reply['content'],
                    $reply['source_number'],
                    $reply['metadata'] ?? []
                );
            }
        }

        return response()->json(['success' => true]);
    }

    private function updateMessageStatus(string $messageId, string $status, ?string $statusCode): void
    {
        // Update in database
        \App\Models\SmsMessage::where('external_id', $messageId)
            ->update([
                'status' => $status,
                'status_code' => $statusCode,
                'delivered_at' => $status === 'delivered' ? now() : null,
            ]);
    }

    private function storeIncomingReply(
        string $replyId,
        string $messageId,
        string $content,
        string $sourceNumber,
        array $metadata = []
    ): void {
        // Store reply in database
        \App\Models\SmsReply::create([
            'external_id' => $replyId,
            'message_id' => $messageId,
            'content' => $content,
            'phone_number' => $sourceNumber,
            'metadata' => $metadata,
        ]);
    }
}
```

### Register Webhook Routes

```php
// routes/api.php

Route::post('/webhooks/messagemedia/delivery', [WebhookController::class, 'handleDeliveryReport']);
Route::post('/webhooks/messagemedia/reply', [WebhookController::class, 'handleReply']);
```

---

## Running Tests

```bash
# Run all tests
php artisan test

# Run specific test class
php artisan test tests/Unit/ClientTest.php

# Run with coverage
php artisan test --coverage

# Run only unit tests
php artisan test tests/Unit

# Run only integration tests
php artisan test --group=integration
```

---

## Debugging

### Enable Debug Logging

```env
MESSAGEMEDIA_DEBUG=true
LOG_CHANNEL=single
```

### Check Logs

```bash
# Tail logs in real-time
tail -f storage/logs/laravel.log | grep MessageMedia
```

### cURL Debug Information

The `HttpClient` class can be extended to log detailed request/response information:

```php
// Extended debugging in Http/HttpClient.php
private function request(string $method, string $url, ?string $body = null): array
{
    // ... existing code ...

    if (config('messagemedia.debug')) {
        Log::debug('MessageMedia Request', [
            'method' => $method,
            'url' => $url,
            'body' => $body,
            'headers' => $headers,
        ]);

        Log::debug('MessageMedia Response', [
            'status_code' => $httpCode,
            'body' => $response,
        ]);
    }

    // ... rest of method ...
}
```

---

## Migration Checklist

```
Pre-Migration:
- [ ] Inventory all MessageMedia SDK usage
- [ ] Create test suite for existing functionality
- [ ] Set up staging environment
- [ ] Backup production data

Migration:
- [ ] Install new package
- [ ] Configure environment variables
- [ ] Update all imports
- [ ] Refactor client initialization
- [ ] Update service classes
- [ ] Test locally

Validation:
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Staging environment tests
- [ ] Load testing (if applicable)

Deployment:
- [ ] Deploy to production
- [ ] Monitor logs for errors
- [ ] Verify message sending
- [ ] Check delivery reports
- [ ] Confirm webhook functionality

Post-Migration:
- [ ] Remove old SDK dependency
- [ ] Document new implementation
- [ ] Train team members
- [ ] Archive migration documentation
```
