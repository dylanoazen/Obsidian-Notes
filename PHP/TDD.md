---
tags: [testing, tdd]
status: draft
---
# TDD em PHP
## Por que isso importa
Red-Green-Refactor: escreve o teste antes, vê falhar, faz passar, refatora.

## Red-Green-Refactor
1. **RED** — escreve teste que FALHA (comportamento desejado)
2. **GREEN** — escreve o MÍNIMO de código pra passar
3. **REFACTOR** — melhora o código sem mudar comportamento

## Na Prática
```php
// 1. RED
public function test_confirm_changes_status(): void {
    $appointment = Appointment::create($doctorId, $patientId, $slot);
    $appointment->confirm();
    $this->assertEquals(Status::Confirmed, $appointment->status());
}
// RODA → FALHA

// 2. GREEN
public function confirm(): void {
    $this->status = Status::Confirmed;
}
// RODA → PASSA

// 3. REFACTOR
public function confirm(): void {
    if ($this->status !== Status::Pending) {
        throw new CannotConfirmException();
    }
    $this->status = Status::Confirmed;
    $this->recordEvent(new AppointmentConfirmed($this->id));
}
```

## Test Doubles
- **Fake**: implementação real simplificada (InMemoryRepo) — **prefira pra domínio**
- **Stub**: retorna valor fixo (`willReturn(...)`)
- **Mock**: verifica se método foi chamado (`expects($this->once())`)
- **Spy**: grava chamadas pra verificar depois

```php
// Fake — testa estado real
$repo = new InMemoryAppointmentRepository();
$service = new BookingService($repo);
$service->book($dto);
$this->assertNotNull($repo->findById($id)); // REALMENTE persistiu?

// Mock — testa interação
$dispatcher = $this->createMock(EventDispatcherInterface::class);
$dispatcher->expects($this->once())->method('dispatch');
```

## Testar Domínio Isolado do Framework
```php
// SEM banco, SEM HTTP, SEM framework — milissegundos
public function test_cannot_book_unavailable_slot(): void {
    $slot = Slot::booked();
    $this->expectException(SlotNotAvailableException::class);
    Appointment::create($doctorId, $patientId, $slot);
}
```

## Perguntas que podem cair
1. "Descreva seu workflow TDD" → Red-Green-Refactor. Teste do comportamento primeiro, implemento o mínimo, refatoro.
2. "Mock vs Fake?" → Mock pra interações. Fake pra estado real. Prefiro fakes pro domínio.
3. "Como testa domínio sem banco?" → Fake repo in-memory. Entity é POPO.

## Links
- [[PHP/TestingEstrategia]] · [[PHP/Testing]] · [[Architecture/DDD]]
