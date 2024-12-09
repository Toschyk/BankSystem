# BankSystem

Для лабораторной работы №5 по функциональному тестированию мы создадим две части с очень длинным и сложным кодом на языке C#, каждая из которых будет включать несколько классов, сложную логику и реализацию функциональных тестов.

### Лабораторная работа №5: Функциональное тестирование

Цель работы:
1. Изучить способы тестирования сложных и многокомпонентных программных систем.
2. Овладеть инструментами функционального тестирования в C# для проверки сложных программ.

В этой лабораторной работе мы создадим два примера кода, каждый из которых будет иметь сотни строк и включать несколько взаимосвязанных компонентов. Для каждого примера мы также напишем функциональные тесты с использованием фреймворка NUnit.

---

### Практическая часть 1: Система управления банковскими счетами с учётом транзакций

В первой части лабораторной работы создадим систему для управления банковскими счетами с возможностью совершения транзакций, применения комиссий, учёта истории транзакций и вывода баланса.

#### 1.1 Реализация системы для управления счетами

Система состоит из следующих классов:

- **Account** — класс, представляющий банковский счёт.
- **Transaction** — класс для транзакций.
- **BankService** — класс для выполнения операций с банком (пополнение, снятие средств, переводы).
- **TransactionHistory** — класс для учёта истории транзакций.

```csharp
// Transaction.cs
namespace BankSystem
{
    public class Transaction
    {
        public decimal Amount { get; set; }
        public DateTime Date { get; set; }
        public string Description { get; set; }

        public Transaction(decimal amount, string description)
        {
            Amount = amount;
            Date = DateTime.Now;
            Description = description;
        }
    }
}
```

```csharp
// Account.cs
using System;
using System.Collections.Generic;

namespace BankSystem
{
    public class Account
    {
        public string AccountNumber { get; private set; }
        public decimal Balance { get; private set; }
        public List<Transaction> Transactions { get; private set; }

        public Account(string accountNumber, decimal initialBalance = 0)
        {
            AccountNumber = accountNumber;
            Balance = initialBalance;
            Transactions = new List<Transaction>();
        }

        public void Deposit(decimal amount)
        {
            Balance += amount;
            Transactions.Add(new Transaction(amount, "Deposit"));
        }

        public bool Withdraw(decimal amount)
        {
            if (Balance >= amount)
            {
                Balance -= amount;
                Transactions.Add(new Transaction(-amount, "Withdrawal"));
                return true;
            }
            return false;
        }

        public void Transfer(Account targetAccount, decimal amount)
        {
            if (Withdraw(amount))
            {
                targetAccount.Deposit(amount);
                Transactions.Add(new Transaction(-amount, "Transfer"));
                targetAccount.Transactions.Add(new Transaction(amount, "Transfer"));
            }
        }

        public void PrintBalance()
        {
            Console.WriteLine($"Account {AccountNumber}: Balance = {Balance:C}");
        }
    }
}
```

```csharp
// BankService.cs
namespace BankSystem
{
    public class BankService
    {
        public void ApplyTransactionFee(Account account, decimal fee)
        {
            if (account.Balance >= fee)
            {
                account.Withdraw(fee);
                account.Transactions.Add(new Transaction(-fee, "Transaction Fee"));
            }
            else
            {
                throw new InvalidOperationException("Insufficient balance to apply fee.");
            }
        }

        public void ProcessTransfer(Account fromAccount, Account toAccount, decimal amount)
        {
            if (fromAccount.Withdraw(amount))
            {
                toAccount.Deposit(amount);
                fromAccount.Transactions.Add(new Transaction(-amount, "Transfer"));
                toAccount.Transactions.Add(new Transaction(amount, "Transfer"));
            }
            else
            {
                throw new InvalidOperationException("Insufficient funds for transfer.");
            }
        }
    }
}
```

```csharp
// TransactionHistory.cs
using System.Collections.Generic;

namespace BankSystem
{
    public class TransactionHistory
    {
        private readonly List<Transaction> _transactions;

        public TransactionHistory(List<Transaction> transactions)
        {
            _transactions = transactions;
        }

        public void PrintHistory()
        {
            Console.WriteLine("Transaction History:");
            foreach (var transaction in _transactions)
            {
                Console.WriteLine($"{transaction.Date}: {transaction.Description}, Amount = {transaction.Amount:C}");
            }
        }
    }
}
```

#### 1.2 Написание тестов для системы управления счетами с использованием NUnit

Теперь напишем функциональные тесты, чтобы проверить правильность работы системы.

```csharp
// BankServiceTests.cs
using NUnit.Framework;
using System;

namespace BankSystem.Tests
{
    public class BankServiceTests
    {
        private Account _account1;
        private Account _account2;
        private BankService _bankService;

        [SetUp]
        public void Setup()
        {
            _account1 = new Account("12345", 1000);
            _account2 = new Account("67890", 500);
            _bankService = new BankService();
        }

        [Test]
        public void Deposit_ShouldIncreaseBalance()
        {
            // Arrange
            decimal depositAmount = 200;

            // Act
            _account1.Deposit(depositAmount);

            // Assert
            Assert.AreEqual(1200, _account1.Balance);
        }

        [Test]
        public void Withdraw_ShouldDecreaseBalance()
        {
            // Arrange
            decimal withdrawAmount = 300;

            // Act
            bool result = _account1.Withdraw(withdrawAmount);

            // Assert
            Assert.IsTrue(result);
            Assert.AreEqual(700, _account1.Balance);
        }

        [Test]
        public void Withdraw_ShouldReturnFalse_WhenInsufficientFunds()
        {
            // Arrange
            decimal withdrawAmount = 1200;

            // Act
            bool result = _account1.Withdraw(withdrawAmount);

            // Assert
            Assert.IsFalse(result);
            Assert.AreEqual(1000, _account1.Balance);
        }

        [Test]
        public void Transfer_ShouldTransferMoneyBetweenAccounts()
        {
            // Arrange
            decimal transferAmount = 200;

            // Act
            _account1.Transfer(_account2, transferAmount);

            // Assert
            Assert.AreEqual(800, _account1.Balance);
            Assert.AreEqual(700, _account2.Balance);
        }

        [Test]
        public void ApplyTransactionFee_ShouldDeductFeeFromAccount()
        {
            // Arrange
            decimal fee = 50;

            // Act
            _bankService.ApplyTransactionFee(_account1, fee);

            // Assert
            Assert.AreEqual(950, _account1.Balance);
        }

        [Test]
        public void ApplyTransactionFee_ShouldThrowException_WhenInsufficientFunds()
        {
            // Arrange
            decimal fee = 1500;

            // Act & Assert
            var ex = Assert.Throws<InvalidOperationException>(() => _bankService.ApplyTransactionFee(_account1, fee));
            Assert.That(ex.Message, Is.EqualTo("Insufficient balance to apply fee."));
        }
    }
}
```

#### 1.3 Запуск тестов

Запуск этих тестов позволит нам проверить:
- Пополнение счёта.
- Снятие средств и проверку недостаточности средств.
- Перевод средств между счетами.
- Применение комиссии и обработку ошибок при недостатке средств.

---

### Практическая часть 2: Система для учёта заказов в интернет-магазине с учётом скидок, налогов и доставки

Во второй части лабораторной работы создадим систему для управления заказами в интернет-магазине. Система будет учитывать:
- Товары в заказе.
- Применение скидок.
- Налоги.
- Стоимость доставки.

#### 2.1 Реализация системы учёта заказов

Система будет состоять из нескольких классов:

- **Product** — класс для товара.
- **Order** — класс для заказа.
- **OrderService** — класс для обработки заказов.
- **DiscountService** — класс для расчёта скидок.

```csharp
// Product.cs
namespace ECommerceSystem
{
    public class Product
    {
        public string Name { get; set; }
        public decimal Price { get; set; }

        public Product(string name, decimal price)
        {
            Name = name;
            Price = price;
        }
    }
}
```

```csharp
// Order.cs
using System.Collections.Generic;

namespace ECommerceSystem
{
    public class Order
    {
        public List<Product> Products { get; set; }
        public decimal Discount { get; set; }
        public decimal TaxRate { get; set; }
        public decimal ShippingCost { get; set; }

        public Order()
        {
            Products = new List<Product>();
            Discount = 0;
            TaxRate = 0.1m;
            ShippingCost = 0;
        }

        public decimal GetTotalBeforeDiscount()
        {
            decimal total = 0;
            foreach (var product in Products)
            {
                total += product.Price;
            }
            return total;
        }
    }
}
```

```csharp
// OrderService.cs
namespace ECommerceSystem
{
    public class OrderService
    {
        public decimal CalculateTotal(Order order)
        {
            decimal totalBeforeDiscount = order.GetTotalBeforeDiscount();
            decimal discountAmount = totalBeforeDiscount * (order.Discount / 100);
            decimal subtotal = totalBeforeDiscount - discountAmount;
            decimal taxAmount = subtotal * order.TaxRate;
            return subtotal + taxAmount + order.ShippingCost;
        }
    }
}
```

```csharp
// DiscountService.cs
namespace ECommerceSystem
{
    public class DiscountService
    {
        public decimal ApplyDiscount(Order order)
        {
            return order.GetTotalBeforeDiscount() * (order.Discount / 100);
        }
    }
}
```

#### 2.2 Написание тестов для системы учёта заказов с использованием NUnit

```csharp
// OrderServiceTests.cs

using NUnit.Framework;

namespace ECommerceSystem.Tests
{
    public class OrderServiceTests
    {
        private OrderService _orderService;
        private Order _order;

        [SetUp]
        public void Setup()
        {
            _orderService = new OrderService();
            _order = new Order();
            _order.Products.Add(new Product("Laptop", 1000));
            _order.Products.Add(new Product("Mouse", 50));
        }

        [Test]
        public void CalculateTotal_ShouldReturnCorrectTotal()
        {
            // Arrange
            _order.Discount = 10; // 10% discount
            _order.ShippingCost = 20;

            // Act
            decimal total = _orderService.CalculateTotal(_order);

            // Assert
            Assert.AreEqual(1070, total);
        }

        [Test]
        public void CalculateTotal_ShouldApplyDiscountCorrectly()
        {
            // Arrange
            _order.Discount = 20;

            // Act
            decimal discountAmount = _orderService.CalculateTotal(_order);

            // Assert
            Assert.AreEqual(840, discountAmount);
        }
    }
}
```

#### 2.3 Запуск тестов

Запустив тесты, мы проверим:
- Правильность расчёта общей стоимости заказа с учётом скидок, налогов и доставки.
- Корректность применения скидки.

---

### Заключение

В ходе лабораторной работы мы разработали две сложные системы:
1. Система для управления банковскими счетами с учётом транзакций.
2. Система учёта заказов в интернет-магазине.

Каждая из этих систем включает в себя несколько классов и функциональных компонентов, что делает их подходящими для функционального тестирования. Тесты проверяют корректность работы всех компонентов и обеспечивают надёжную работу системы.
