# WION Platform Specification

# Billing Service

## Wi Credit Platform

**Version:** 1.0

**Status:** Draft

**Author:** WION Platform Team

---

# 1. Executive Summary

## 1.1 Background

WION Platform đang phát triển theo mô hình SaaS đa sản phẩm, bao gồm:

* WION POS
* WION FnB
* WION SPA
* WIPIX
* AI Services
* Platform Services

Các sản phẩm này đều phát sinh nhu cầu thanh toán hoặc tiêu thụ tài nguyên như:

* AI Generation
* OCR
* SMS
* Storage
* Marketplace
* Add-on
* Resource Package

Hiện tại mỗi dịch vụ đang có cách tính phí riêng, chưa có một nền tảng thanh toán thống nhất.

Điều này gây khó khăn trong việc:

* Quản lý số dư
* Theo dõi lịch sử giao dịch
* Đối soát
* Xuất hóa đơn điện tử
* Tích hợp giữa các sản phẩm

---

## 1.2 Objective

Xây dựng một Billing Platform dùng chung cho toàn bộ hệ sinh thái WION.

Billing Service sẽ cung cấp:

* Ví điện tử nội bộ (Wallet)
* Wi Credit
* Payment Gateway Integration
* Credit Consumption
* Ledger
* Invoice Integration
* SDK cho toàn bộ Platform

Billing Service KHÔNG thay thế Payment Gateway.

Billing Service KHÔNG quản lý hóa đơn điện tử.

---

# 2. Business Goals

Mục tiêu:

✓ Một ví dùng chung cho toàn Platform

✓ Một loại tài sản thanh toán thống nhất

✓ Một API dùng chung cho mọi sản phẩm

✓ Một lịch sử giao dịch tập trung

✓ Có thể mở rộng Marketplace

✓ Có thể mở rộng Billing theo Subscription

---

# 3. Business Scope

## In Scope

### Wallet

* Quản lý số dư

* Wi Credit

* Khóa số dư (Reserved)

---

### Payment

* Nạp tiền

* QR Payment

* Payment Webhook

* Payment History

---

### Credit

* Consume

* Refund

* Adjustment

* Transfer (Future)

---

### Ledger

* Double-entry Transaction

* Audit Log

---

### Invoice

* Tích hợp Invoice Hub

* Mapping Payment → Invoice

---

### SDK

* Consume Credit

* Check Balance

* Refund Credit

---

# 4. Out of Scope

Không bao gồm:

* Kế toán

* Thuế

* Quản lý hóa đơn

* ERP

* Ngân hàng

* Ví điện tử bên thứ ba

---

# 5. Business Concepts

## Wallet

Một Tenant có một ví.

Wallet lưu:

* Balance

* Reserved Balance

* Currency

---

## Asset

Wallet có thể chứa nhiều loại tài sản.

Ví dụ:

Wi Credit

Promotion Credit

Gift Credit

Trial Credit

AI Token (Future)

---

## Wi Credit

Wi Credit là đơn vị thanh toán nội bộ của WION Platform.

Nguyên tắc:

1 VNĐ = 1 Wi Credit (có thể cấu hình)

Không quy đổi ngược.

Không chuyển ra tiền mặt.

Có thể tiêu thụ tại mọi sản phẩm WION.

---

## Transaction

Mọi thay đổi số dư đều phải sinh Transaction.

Không tồn tại việc cập nhật Balance trực tiếp.

---

## Ledger

Ledger là nguồn dữ liệu chuẩn phục vụ:

* Audit

* Đối soát

* Truy vết

---

# 6. Actors

Platform Admin

Tenant Owner

Finance

Product Service

Payment Gateway

Invoice Hub

---

# 7. Functional Requirements

## FR-01

Nạp Credit

...

## FR-02

Thanh toán QR

...

## FR-03

Webhook

...

## FR-04

Cộng Credit

...

## FR-05

Trừ Credit

...

## FR-06

Hoàn Credit

...

## FR-07

Lịch sử giao dịch

...

## FR-08

Xuất hóa đơn

...

## FR-09

Kiểm tra số dư

...

## FR-10

SDK

...

---

# 8. Business Rules

Ví dụ:

BR-001

Balance không được âm.

BR-002

Mọi Consume phải tạo Ledger.

BR-003

Invoice chỉ được tạo khi Payment thành công.

BR-004

Webhook phải Idempotent.

BR-005

ReferenceId phải duy nhất theo Source.

...

---

# 9. Use Cases

UC-01

Topup Credit

UC-02

Consume AI Credit

UC-03

Refund

UC-04

Issue Invoice

UC-05

Check Balance

UC-06

Transaction History

---

# 10. User Journey

Topup

Portal

↓

Billing

↓

TPayGate

↓

QR

↓

Webhook

↓

Billing

↓

Wallet

↓

Invoice Hub

---

# 11. Domain Model

Wallet

WalletAsset

Payment

PaymentMethod

PaymentTransaction

Ledger

CreditTransaction

InvoiceReference

Pricing

ConsumptionRecord

---

# 12. API Specification

Wallet APIs

Payment APIs

Invoice APIs

SDK APIs

Webhook APIs

---

# 13. Event Contract

Publish

PaymentSucceeded

PaymentFailed

CreditAdded

CreditConsumed

CreditRefunded

BalanceChanged

InvoiceIssued

Consume

TenantDeleted

SubscriptionExpired

---

# 14. Permission Matrix

Platform Admin

Finance

Tenant Owner

Application

SDK

---

# 15. Non-functional Requirements

Availability

Audit

Idempotency

Consistency

Performance

Security

Observability

---

# 16. Future Roadmap

Phase 1

Wallet

Payment

Credit

Invoice

SDK

Phase 2

Promotion Credit

Gift Credit

Marketplace

Subscription Billing

Phase 3

Usage Billing

AI Billing

Storage Billing

Marketplace Settlement

Phase 4

Partner Revenue Sharing

Coupon Engine

Billing Dashboard
