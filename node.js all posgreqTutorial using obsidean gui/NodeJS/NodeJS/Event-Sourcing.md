# Event Sourcing in Node.js

A comprehensive guide to implementing Event Sourcing pattern in Node.js applications.

## 1. Event Store

### Event Store Implementation
```javascript
class Event {
  constructor(type, data, metadata = {}) {
    this.type = type;
    this.data = data;
    this.metadata = {
      timestamp: new Date().toISOString(),
      ...metadata
    };
  }
}

class EventStore {
  constructor(options = {}) {
    this.events = [];
    this.subscribers = new Map();
    this.snapshots = new Map();
    this.snapshotFrequency = options.snapshotFrequency || 100;
  }

  async append(streamId, events, expectedVersion = null) {
    if (expectedVersion !== null) {
      const currentVersion = await this.getStreamVersion(streamId);
      if (currentVersion !== expectedVersion) {
        throw new Error('Concurrency conflict');
      }
    }
    
    const streamEvents = Array.isArray(events) ? events : [events];
    const startVersion = await this.getStreamVersion(streamId);
    
    for (let i = 0; i < streamEvents.length; i++) {
      const event = streamEvents[i];
      const eventRecord = {
        streamId,
        version: startVersion + i + 1,
        event
      };
      
      this.events.push(eventRecord);
      await this.notifySubscribers(event);
    }
    
    // Check if snapshot needed
    const newVersion = startVersion + streamEvents.length;
    if (newVersion % this.snapshotFrequency === 0) {
      await this.createSnapshot(streamId);
    }
    
    return newVersion;
  }

  async getEvents(streamId, fromVersion = 0) {
    return this.events
      .filter(e => e.streamId === streamId && e.version > fromVersion)
      .map(e => e.event);
  }

  async getStreamVersion(streamId) {
    const streamEvents = this.events.filter(e => e.streamId === streamId);
    return streamEvents.length;
  }

  subscribe(eventType, handler) {
    if (!this.subscribers.has(eventType)) {
      this.subscribers.set(eventType, new Set());
    }
    this.subscribers.get(eventType).add(handler);
  }

  async notifySubscribers(event) {
    const handlers = this.subscribers.get(event.type) || new Set();
    for (const handler of handlers) {
      await handler(event);
    }
  }

  async createSnapshot(streamId) {
    const aggregate = await this.loadAggregate(streamId);
    this.snapshots.set(streamId, {
      version: await this.getStreamVersion(streamId),
      state: aggregate.getState()
    });
  }

  async getSnapshot(streamId) {
    return this.snapshots.get(streamId);
  }
}
```

## 2. Aggregate Implementation

### Event-Sourced Aggregate
```javascript
class EventSourcedAggregate {
  constructor(id) {
    this.id = id;
    this.version = 0;
    this.changes = [];
  }

  getUncommittedChanges() {
    return this.changes;
  }

  markChangesAsCommitted() {
    this.changes = [];
  }

  loadFromHistory(events) {
    for (const event of events) {
      this.applyChange(event, false);
    }
  }

  applyChange(event, isNew = true) {
    const handler = this.getEventHandler(event);
    if (handler) {
      handler.call(this, event);
    }
    
    if (isNew) {
      this.changes.push(event);
    }
    
    this.version++;
  }

  getEventHandler(event) {
    const handlerName = `apply${event.type}`;
    return this[handlerName];
  }
}

class BankAccount extends EventSourcedAggregate {
  constructor(id) {
    super(id);
    this.balance = 0;
    this.status = 'new';
  }

  create(initialBalance) {
    if (this.status !== 'new') {
      throw new Error('Account already created');
    }
    
    this.applyChange(new Event('AccountCreated', {
      accountId: this.id,
      initialBalance
    }));
  }

  deposit(amount) {
    if (amount <= 0) {
      throw new Error('Amount must be positive');
    }
    
    this.applyChange(new Event('MoneyDeposited', {
      accountId: this.id,
      amount
    }));
  }

  withdraw(amount) {
    if (amount <= 0) {
      throw new Error('Amount must be positive');
    }
    
    if (this.balance < amount) {
      throw new Error('Insufficient funds');
    }
    
    this.applyChange(new Event('MoneyWithdrawn', {
      accountId: this.id,
      amount
    }));
  }

  applyAccountCreated(event) {
    this.balance = event.data.initialBalance;
    this.status = 'active';
  }

  applyMoneyDeposited(event) {
    this.balance += event.data.amount;
  }

  applyMoneyWithdrawn(event) {
    this.balance -= event.data.amount;
  }

  getState() {
    return {
      id: this.id,
      balance: this.balance,
      status: this.status,
      version: this.version
    };
  }
}
```

## 3. Repository Implementation

### Event-Sourced Repository
```javascript
class EventSourcedRepository {
  constructor(eventStore, aggregateType) {
    this.eventStore = eventStore;
    this.aggregateType = aggregateType;
  }

  async save(aggregate) {
    const changes = aggregate.getUncommittedChanges();
    if (changes.length === 0) {
      return;
    }
    
    await this.eventStore.append(
      aggregate.id,
      changes,
      aggregate.version - changes.length
    );
    
    aggregate.markChangesAsCommitted();
  }

  async getById(id) {
    const snapshot = await this.eventStore.getSnapshot(id);
    const fromVersion = snapshot ? snapshot.version : 0;
    
    const events = await this.eventStore.getEvents(id, fromVersion);
    if (events.length === 0 && !snapshot) {
      return null;
    }
    
    const aggregate = new this.aggregateType(id);
    if (snapshot) {
      aggregate.loadFromHistory([new Event('Snapshot', snapshot.state)]);
    }
    
    aggregate.loadFromHistory(events);
    return aggregate;
  }
}
```

## 4. Event Handlers

### Event Handler Implementation
```javascript
class EventHandler {
  constructor(eventStore) {
    this.eventStore = eventStore;
  }

  async handle(event) {
    const handlerName = `handle${event.type}`;
    if (this[handlerName]) {
      await this[handlerName](event);
    }
  }
}

class AccountEventHandler extends EventHandler {
  constructor(eventStore, readModel) {
    super(eventStore);
    this.readModel = readModel;
  }

  async handleAccountCreated(event) {
    await this.readModel.createAccount(
      event.data.accountId,
      event.data.initialBalance
    );
  }

  async handleMoneyDeposited(event) {
    await this.readModel.updateBalance(
      event.data.accountId,
      event.data.amount,
      'credit'
    );
  }

  async handleMoneyWithdrawn(event) {
    await this.readModel.updateBalance(
      event.data.accountId,
      event.data.amount,
      'debit'
    );
  }
}
```

## 5. Read Model

### Read Model Implementation
```javascript
class ReadModel {
  constructor() {
    this.accounts = new Map();
  }

  async createAccount(accountId, balance) {
    this.accounts.set(accountId, {
      id: accountId,
      balance,
      transactions: []
    });
  }

  async updateBalance(accountId, amount, type) {
    const account = this.accounts.get(accountId);
    if (!account) {
      throw new Error('Account not found');
    }
    
    account.balance = type === 'credit' 
      ? account.balance + amount
      : account.balance - amount;
    
    account.transactions.push({
      type,
      amount,
      timestamp: new Date().toISOString()
    });
  }

  async getAccount(accountId) {
    return this.accounts.get(accountId);
  }

  async getTransactions(accountId) {
    const account = await this.getAccount(accountId);
    return account ? account.transactions : [];
  }
}
```

## 6. Application Service

### Application Service Implementation
```javascript
class BankAccountService {
  constructor(repository, eventStore) {
    this.repository = repository;
    this.eventStore = eventStore;
  }

  async createAccount(accountId, initialBalance) {
    const account = new BankAccount(accountId);
    account.create(initialBalance);
    await this.repository.save(account);
    return account;
  }

  async deposit(accountId, amount) {
    const account = await this.repository.getById(accountId);
    if (!account) {
      throw new Error('Account not found');
    }
    
    account.deposit(amount);
    await this.repository.save(account);
    return account;
  }

  async withdraw(accountId, amount) {
    const account = await this.repository.getById(accountId);
    if (!account) {
      throw new Error('Account not found');
    }
    
    account.withdraw(amount);
    await this.repository.save(account);
    return account;
  }

  async getBalance(accountId) {
    const account = await this.repository.getById(accountId);
    if (!account) {
      throw new Error('Account not found');
    }
    
    return account.balance;
  }
}
```

## Related Topics
- [[DDD-Implementation]] - Domain-Driven Design
- [[CQRS-Advanced]] - Advanced CQRS
- [[System-Architecture]] - System Architecture
- [[Enterprise-Patterns]] - Enterprise Patterns

## Practice Projects
1. Build banking system with event sourcing
2. Implement inventory management
3. Create order processing system
4. Design audit system

## Resources
- [Event Sourcing Pattern](https://martinfowler.com/eaaDev/EventSourcing.html)
- [[Learning-Resources#EventSourcing|Event Sourcing Resources]]

## Tags
#event-sourcing #nodejs #architecture #ddd #cqrs
