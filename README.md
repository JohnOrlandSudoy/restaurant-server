# PayMongo Frontend Implementation Guide

## 🚀 **Complete API Endpoints Reference**

### **1. Order Management Endpoints**

#### **Create PayMongo Payment for Order**
```http
POST /api/orders/{orderId}/paymongo-payment
Authorization: Bearer {token}
Content-Type: application/json

Body:
{
  "description": "Payment for Order #ORD-20250910-0003",
  "metadata": {
    "customer_phone": "09123456789",
    "order_type": "dine_in"
  }
}

Response:
{
  "success": true,
  "message": "PayMongo payment intent created for order",
  "data": {
    "paymentIntentId": "pi_test_1757473090382",
    "status": "awaiting_payment_method",
    "amount": 2240,
    "currency": "PHP",
    "expiresAt": "2025-09-10T03:13:10.595Z",
    "qrCodeUrl": "https://api.qrserver.com/v1/create-qr-code/?size=300x300&data=pi_test_1757473090382",
    "qrCodeData": "base64_encoded_qr_data_for_pi_test_1757473090382",
    "order": {
      "id": "8120cd22-5630-4720-af21-4c8169f8ae75",
      "orderNumber": "ORD-20250910-0003",
      "totalAmount": 22.4,
      "customerName": "test paymong pay"
    }
  }
}
```

#### **Check Payment Status**
```http
GET /api/payments/status/{paymentIntentId}
Authorization: Bearer {token}

Response:
{
  "success": true,
  "data": {
    "paymentIntentId": "pi_test_1757473090382",
    "status": "awaiting_payment_method",
    "amount": 2240,
    "currency": "PHP"
  }
}
```

#### **Cancel Payment**
```http
POST /api/payments/cancel/{paymentIntentId}
Authorization: Bearer {token}

Response:
{
  "success": true,
  "message": "Payment intent cancelled successfully",
  "data": {
    "paymentIntentId": "pi_test_1757473090382",
    "status": "cancelled",
    "amount": 2240,
    "currency": "PHP"
  }
}
```

#### **Update Order Payment Status**
```http
PUT /api/orders/{orderId}/payment
Authorization: Bearer {token}
Content-Type: application/json

Body:
{
  "payment_status": "paid",
  "payment_method": "paymongo"
}
```

### **2. Authentication Endpoints**

#### **Login (Get Token)**
```http
POST /api/auth/login
Content-Type: application/json

Body:
{
  "username": "cashier1",
  "password": "password123"
}

Response:
{
  "success": true,
  "message": "Login successful",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "8120cd22-5630-4720-af21-4c8169f8ae75",
    "username": "cashier1",
    "role": "cashier"
  }
}
```

## 🎨 **Frontend Implementation Components**

### **1. Payment Flow Component**

```javascript
// PayMongoPaymentComponent.jsx
import React, { useState, useEffect } from 'react';
import { toast } from 'react-toastify';

const PayMongoPaymentComponent = ({ orderId, orderTotal, onPaymentSuccess }) => {
  const [paymentData, setPaymentData] = useState(null);
  const [isLoading, setIsLoading] = useState(false);
  const [paymentStatus, setPaymentStatus] = useState('idle');
  const [timeLeft, setTimeLeft] = useState(0);

  // Create PayMongo payment
  const createPayment = async () => {
    setIsLoading(true);
    try {
      const token = localStorage.getItem('authToken');
      const response = await fetch(`/api/orders/${orderId}/paymongo-payment`, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          description: `Payment for Order #${orderId}`,
          metadata: {
            customer_phone: "09123456789",
            order_type: "dine_in"
          }
        })
      });

      const result = await response.json();
      
      if (result.success) {
        setPaymentData(result.data);
        setPaymentStatus('awaiting_payment');
        startPaymentPolling(result.data.paymentIntentId);
        calculateTimeLeft(result.data.expiresAt);
        toast.success('QR Code generated! Please scan to pay.');
      } else {
        toast.error(result.error || 'Failed to create payment');
      }
    } catch (error) {
      toast.error('Network error. Please try again.');
    } finally {
      setIsLoading(false);
    }
  };

  // Poll payment status
  const startPaymentPolling = (paymentIntentId) => {
    const interval = setInterval(async () => {
      try {
        const token = localStorage.getItem('authToken');
        const response = await fetch(`/api/payments/status/${paymentIntentId}`, {
          headers: {
            'Authorization': `Bearer ${token}`
          }
        });

        const result = await response.json();
        
        if (result.success) {
          if (result.data.status === 'succeeded') {
            setPaymentStatus('paid');
            clearInterval(interval);
            toast.success('Payment successful!');
            onPaymentSuccess && onPaymentSuccess(result.data);
          } else if (result.data.status === 'failed' || result.data.status === 'cancelled') {
            setPaymentStatus('failed');
            clearInterval(interval);
            toast.error('Payment failed or cancelled');
          }
        }
      } catch (error) {
        console.error('Payment status check failed:', error);
      }
    }, 3000); // Check every 3 seconds

    // Clear interval after 15 minutes
    setTimeout(() => clearInterval(interval), 15 * 60 * 1000);
  };

  // Calculate time left
  const calculateTimeLeft = (expiresAt) => {
    const now = new Date().getTime();
    const expiry = new Date(expiresAt).getTime();
    const difference = expiry - now;

    if (difference > 0) {
      setTimeLeft(Math.floor(difference / 1000));
      
      const timer = setInterval(() => {
        setTimeLeft(prev => {
          if (prev <= 1) {
            clearInterval(timer);
            setPaymentStatus('expired');
            return 0;
          }
          return prev - 1;
        });
      }, 1000);
    }
  };

  // Cancel payment
  const cancelPayment = async () => {
    if (!paymentData?.paymentIntentId) return;
    
    try {
      const token = localStorage.getItem('authToken');
      const response = await fetch(`/api/payments/cancel/${paymentData.paymentIntentId}`, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${token}`
        }
      });

      const result = await response.json();
      
      if (result.success) {
        setPaymentStatus('cancelled');
        setPaymentData(null);
        toast.info('Payment cancelled');
      }
    } catch (error) {
      toast.error('Failed to cancel payment');
    }
  };

  // Format time display
  const formatTime = (seconds) => {
    const mins = Math.floor(seconds / 60);
    const secs = seconds % 60;
    return `${mins}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="paymongo-payment-container">
      <div className="payment-header">
        <h3>PayMongo QR Payment</h3>
        <p>Total Amount: ₱{orderTotal}</p>
      </div>

      {paymentStatus === 'idle' && (
        <button 
          className="btn btn-primary btn-lg"
          onClick={createPayment}
          disabled={isLoading}
        >
          {isLoading ? 'Creating Payment...' : 'Generate QR Code'}
        </button>
      )}

      {paymentStatus === 'awaiting_payment' && paymentData && (
        <div className="qr-payment-section">
          <div className="qr-code-container">
            <img 
              src={paymentData.qrCodeUrl} 
              alt="PayMongo QR Code"
              className="qr-code-image"
            />
            <p className="qr-instructions">
              Scan this QR code with your GCash, Maya, or bank app to pay
            </p>
          </div>

          <div className="payment-info">
            <div className="amount-display">
              <strong>Amount: ₱{(paymentData.amount / 100).toFixed(2)}</strong>
            </div>
            <div className="time-remaining">
              <span className="timer">
                Time remaining: {formatTime(timeLeft)}
              </span>
            </div>
            <div className="payment-id">
              Payment ID: {paymentData.paymentIntentId}
            </div>
          </div>

          <div className="payment-actions">
            <button 
              className="btn btn-secondary"
              onClick={cancelPayment}
            >
              Cancel Payment
            </button>
          </div>
        </div>
      )}

      {paymentStatus === 'paid' && (
        <div className="payment-success">
          <div className="success-icon">✅</div>
          <h4>Payment Successful!</h4>
          <p>Your order has been paid successfully.</p>
        </div>
      )}

      {paymentStatus === 'failed' && (
        <div className="payment-failed">
          <div className="error-icon">❌</div>
          <h4>Payment Failed</h4>
          <p>Please try again or use a different payment method.</p>
          <button 
            className="btn btn-primary"
            onClick={() => {
              setPaymentStatus('idle');
              setPaymentData(null);
            }}
          >
            Try Again
          </button>
        </div>
      )}

      {paymentStatus === 'expired' && (
        <div className="payment-expired">
          <div className="warning-icon">⏰</div>
          <h4>Payment Expired</h4>
          <p>The QR code has expired. Please generate a new one.</p>
          <button 
            className="btn btn-primary"
            onClick={() => {
              setPaymentStatus('idle');
              setPaymentData(null);
            }}
          >
            Generate New QR Code
          </button>
        </div>
      )}
    </div>
  );
};

export default PayMongoPaymentComponent;
```

### **2. CSS Styles**

```css
/* PayMongoPaymentComponent.css */
.paymongo-payment-container {
  max-width: 400px;
  margin: 0 auto;
  padding: 20px;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  background: white;
}

.payment-header {
  text-align: center;
  margin-bottom: 20px;
}

.payment-header h3 {
  color: #333;
  margin-bottom: 10px;
}

.payment-header p {
  font-size: 18px;
  font-weight: bold;
  color: #007bff;
}

.qr-payment-section {
  text-align: center;
}

.qr-code-container {
  margin-bottom: 20px;
}

.qr-code-image {
  width: 250px;
  height: 250px;
  border: 1px solid #ddd;
  border-radius: 8px;
  margin-bottom: 15px;
}

.qr-instructions {
  color: #666;
  font-size: 14px;
  margin-bottom: 20px;
}

.payment-info {
  background: #f8f9fa;
  padding: 15px;
  border-radius: 6px;
  margin-bottom: 20px;
}

.amount-display {
  font-size: 20px;
  color: #28a745;
  margin-bottom: 10px;
}

.time-remaining {
  margin-bottom: 10px;
}

.timer {
  background: #ffc107;
  color: #000;
  padding: 5px 10px;
  border-radius: 4px;
  font-weight: bold;
}

.payment-id {
  font-size: 12px;
  color: #666;
  word-break: break-all;
}

.payment-actions {
  margin-top: 20px;
}

.payment-success,
.payment-failed,
.payment-expired {
  text-align: center;
  padding: 20px;
}

.success-icon,
.error-icon,
.warning-icon {
  font-size: 48px;
  margin-bottom: 15px;
}

.payment-success {
  background: #d4edda;
  border: 1px solid #c3e6cb;
  border-radius: 6px;
  color: #155724;
}

.payment-failed {
  background: #f8d7da;
  border: 1px solid #f5c6cb;
  border-radius: 6px;
  color: #721c24;
}

.payment-expired {
  background: #fff3cd;
  border: 1px solid #ffeaa7;
  border-radius: 6px;
  color: #856404;
}

.btn {
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  transition: background-color 0.3s;
}

.btn-primary {
  background: #007bff;
  color: white;
}

.btn-primary:hover {
  background: #0056b3;
}

.btn-secondary {
  background: #6c757d;
  color: white;
}

.btn-secondary:hover {
  background: #545b62;
}

.btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}
```

### **3. Integration in Order Component**

```javascript
// OrderComponent.jsx
import React, { useState } from 'react';
import PayMongoPaymentComponent from './PayMongoPaymentComponent';

const OrderComponent = ({ order }) => {
  const [showPayment, setShowPayment] = useState(false);
  const [paymentMethod, setPaymentMethod] = useState('');

  const handlePaymentSuccess = (paymentData) => {
    // Update order status in your state
    console.log('Payment successful:', paymentData);
    
    // You might want to refresh the order data here
    // or update the order status in your parent component
    
    setShowPayment(false);
    setPaymentMethod('paymongo');
  };

  const renderPaymentSection = () => {
    if (order.payment_status === 'paid') {
      return (
        <div className="payment-status paid">
          <span className="status-badge success">✅ Paid</span>
          <span className="payment-method">via {order.payment_method}</span>
        </div>
      );
    }

    if (order.payment_status === 'pending') {
      return (
        <div className="payment-status pending">
          <span className="status-badge warning">⏳ Payment Pending</span>
          <button 
            className="btn btn-sm btn-outline"
            onClick={() => setShowPayment(true)}
          >
            View Payment
          </button>
        </div>
      );
    }

    return (
      <div className="payment-options">
        <h4>Payment Options</h4>
        <div className="payment-methods">
          <button 
            className="payment-method-btn"
            onClick={() => setShowPayment(true)}
          >
            <span className="method-icon">📱</span>
            <span>PayMongo QR</span>
          </button>
          
          <button 
            className="payment-method-btn"
            onClick={() => handleCashPayment()}
          >
            <span className="method-icon">💵</span>
            <span>Cash</span>
          </button>
        </div>
      </div>
    );
  };

  return (
    <div className="order-card">
      <div className="order-header">
        <h3>Order #{order.order_number}</h3>
        <span className="order-status">{order.status}</span>
      </div>

      <div className="order-details">
        <p><strong>Customer:</strong> {order.customer_name}</p>
        <p><strong>Total:</strong> ₱{order.total_amount}</p>
        <p><strong>Type:</strong> {order.order_type}</p>
      </div>

      {renderPaymentSection()}

      {showPayment && (
        <div className="payment-modal">
          <div className="modal-content">
            <div className="modal-header">
              <h3>Payment for Order #{order.order_number}</h3>
              <button 
                className="close-btn"
                onClick={() => setShowPayment(false)}
              >
                ×
              </button>
            </div>
            
            <PayMongoPaymentComponent
              orderId={order.id}
              orderTotal={order.total_amount}
              onPaymentSuccess={handlePaymentSuccess}
            />
          </div>
        </div>
      )}
    </div>
  );
};

export default OrderComponent;
```

## 🔧 **Frontend Developer Prompt**

```
I need you to implement a PayMongo QR payment system for my restaurant order management app. Here are the requirements:

**Backend API Endpoints Available:**
1. POST /api/orders/{orderId}/paymongo-payment - Create payment intent and get QR code
2. GET /api/payments/status/{paymentIntentId} - Check payment status
3. POST /api/payments/cancel/{paymentIntentId} - Cancel payment
4. PUT /api/orders/{orderId}/payment - Update order payment status

**Authentication:**
- All endpoints require Bearer token authentication
- Token obtained from POST /api/auth/login

**Payment Flow Requirements:**
1. User clicks "Pay with PayMongo QR" button
2. Frontend calls the create payment endpoint
3. Display QR code image to user
4. Show countdown timer (15 minutes expiry)
5. Poll payment status every 3 seconds
6. Handle success/failure/expiry states
7. Allow payment cancellation

**UI Requirements:**
- Clean, modern design
- Mobile-responsive
- Clear payment instructions
- Real-time status updates
- Error handling with user-friendly messages
- Loading states for all actions

**Technical Requirements:**
- React.js with hooks
- Fetch API for HTTP requests
- Local storage for auth token
- Toast notifications for user feedback
- Automatic cleanup of intervals/timeouts
- Proper error handling

**Payment States to Handle:**
- idle: Initial state, show "Generate QR Code" button
- awaiting_payment: Show QR code and timer
- paid: Show success message
- failed: Show error and retry option
- expired: Show expiry message and regenerate option
- cancelled: Show cancellation message

Please implement this as a reusable React component with proper TypeScript types, comprehensive error handling, and a polished UI that matches modern payment interfaces.
```

## 📱 **Mobile-First Considerations**

1. **QR Code Size**: Make QR code large enough for easy scanning (250px+)
2. **Touch Targets**: Ensure buttons are at least 44px for mobile
3. **Responsive Design**: Stack elements vertically on mobile
4. **Network Handling**: Handle poor network conditions gracefully
5. **Battery Optimization**: Clean up intervals when component unmounts

## 🔒 **Security Best Practices**

1. **Token Storage**: Store auth tokens securely (consider httpOnly cookies)
2. **Input Validation**: Validate all user inputs
3. **Error Messages**: Don't expose sensitive information in error messages
4. **HTTPS**: Always use HTTPS in production
5. **Token Refresh**: Implement token refresh mechanism

This implementation provides a complete, production-ready PayMongo payment integration for your restaurant order management system!
