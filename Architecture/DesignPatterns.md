---
tags: [patterns]
status: draft
---
# Design Patterns Backend

## Repository — [[PHP/DoctrineORM]]
## Observer — [[Architecture/DomainEvents]]

## Strategy
```php
interface PricingStrategy { public function calculate(Appointment $a): Money; }
class StandardPricing implements PricingStrategy { }
class InsurancePricing implements PricingStrategy { }
// Container injeta a strategy certa
```

## Factory
```php
class AppointmentFactory {
    public static function fromRequest(BookingRequest $req, Slot $slot): Appointment {
        return new Appointment($req->doctorId, $req->patientId, $slot);
    }
}
```

## Decorator
```php
class TimestampDecorator implements Logger {
    public function __construct(private Logger $inner) {}
    public function log(string $msg): void {
        $this->inner->log('[' . date('c') . '] ' . $msg);
    }
}
```

## Links
- [[PHP/SOLID]] · [[Architecture/DDD]]
