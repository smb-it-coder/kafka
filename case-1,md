To implement a **microservices architecture** for your scenario using **Node.js**, **Kafka**, and **MongoDB**, we’ll break it into manageable steps.

---

## **Architecture Overview**

1. **Services**:
   - **Product Service**:
     - Handles product-related actions (add, modify products).
   - **Order Service**:
     - Processes orders and communicates with the product and user services.
   - **User Service**:
     - Manages customer information like shipping and billing addresses.

2. **Kafka Topics**:
   - `product-events`: Events for product creation or modification.
   - `order-events`: Events for order placement.
   - `user-events`: Events for user-related activities (e.g., address updates).

3. **Database**:
   - Each service has its own **MongoDB** database for data persistence.

4. **Event Flow**:
   - **Product Service** publishes events to `product-events` after adding or modifying a product.
   - **Order Service** consumes `product-events` to ensure product availability.
   - **Order Service** publishes `order-events` to notify the User Service.
   - **User Service** consumes `order-events` to process shipping and billing information.

---

### **Step-by-Step Implementation**

#### **Step 1: Setup Kafka and MongoDB**
1. **Kafka Setup**:
   Use the `docker-compose.yml` file we discussed earlier to start Kafka.
2. **MongoDB Setup**:
   Add MongoDB services to your Docker Compose file:
   ```yaml
   version: '3.8'
   services:
     mongo:
       image: mongo
       container_name: mongo
       ports:
         - "27017:27017"
   ```

Start Kafka and MongoDB using:
```bash
docker-compose up -d
```

---

#### **Step 2: Create Product Service**
1. Install dependencies:
   ```bash
   npm install express mongoose kafkajs body-parser
   ```

2. Create `productService.js`:
   ```javascript
   const express = require('express');
   const mongoose = require('mongoose');
   const { Kafka } = require('kafkajs');

   const app = express();
   app.use(express.json());

   const kafka = new Kafka({ clientId: 'product-service', brokers: ['localhost:9092'] });
   const producer = kafka.producer();
   const topic = 'product-events';

   mongoose.connect('mongodb://localhost:27017/product-service', {
     useNewUrlParser: true,
     useUnifiedTopology: true,
   });

   const ProductSchema = new mongoose.Schema({
     name: String,
     price: Number,
     stock: Number,
   });
   const Product = mongoose.model('Product', ProductSchema);

   app.post('/product', async (req, res) => {
     const { name, price, stock } = req.body;
     const product = await Product.create({ name, price, stock });

     await producer.connect();
     await producer.send({
       topic,
       messages: [{ value: JSON.stringify({ type: 'ADD_PRODUCT', product }) }],
     });
     res.status(201).send(product);
   });

   app.put('/product/:id', async (req, res) => {
     const { id } = req.params;
     const { name, price, stock } = req.body;

     const product = await Product.findByIdAndUpdate(id, { name, price, stock }, { new: true });

     await producer.connect();
     await producer.send({
       topic,
       messages: [{ value: JSON.stringify({ type: 'UPDATE_PRODUCT', product }) }],
     });
     res.status(200).send(product);
   });

   app.listen(3001, () => console.log('Product Service running on port 3001'));
   ```

---

#### **Step 3: Create Order Service**
1. Install dependencies:
   ```bash
   npm install express mongoose kafkajs
   ```

2. Create `orderService.js`:
   ```javascript
   const express = require('express');
   const mongoose = require('mongoose');
   const { Kafka } = require('kafkajs');

   const app = express();
   app.use(express.json());

   const kafka = new Kafka({ clientId: 'order-service', brokers: ['localhost:9092'] });
   const consumer = kafka.consumer({ groupId: 'order-group' });
   const producer = kafka.producer();
   const productTopic = 'product-events';
   const orderTopic = 'order-events';

   mongoose.connect('mongodb://localhost:27017/order-service', {
     useNewUrlParser: true,
     useUnifiedTopology: true,
   });

   const OrderSchema = new mongoose.Schema({
     productId: String,
     userId: String,
     quantity: Number,
     status: String,
   });
   const Order = mongoose.model('Order', OrderSchema);

   const runConsumer = async () => {
     await consumer.connect();
     await consumer.subscribe({ topic: productTopic, fromBeginning: true });

     await consumer.run({
       eachMessage: async ({ topic, partition, message }) => {
         console.log(`Received message: ${message.value.toString()}`);
         // Handle product events (e.g., stock updates)
       },
     });
   };
   runConsumer();

   app.post('/order', async (req, res) => {
     const { productId, userId, quantity } = req.body;

     const order = await Order.create({ productId, userId, quantity, status: 'PENDING' });

     await producer.connect();
     await producer.send({
       topic: orderTopic,
       messages: [{ value: JSON.stringify({ type: 'PLACE_ORDER', order }) }],
     });

     res.status(201).send(order);
   });

   app.listen(3002, () => console.log('Order Service running on port 3002'));
   ```

---

#### **Step 4: Create User Service**
1. Install dependencies:
   ```bash
   npm install express mongoose kafkajs
   ```

2. Create `userService.js`:
   ```javascript
   const express = require('express');
   const mongoose = require('mongoose');
   const { Kafka } = require('kafkajs');

   const app = express();
   app.use(express.json());

   const kafka = new Kafka({ clientId: 'user-service', brokers: ['localhost:9092'] });
   const consumer = kafka.consumer({ groupId: 'user-group' });
   const orderTopic = 'order-events';

   mongoose.connect('mongodb://localhost:27017/user-service', {
     useNewUrlParser: true,
     useUnifiedTopology: true,
   });

   const UserSchema = new mongoose.Schema({
     name: String,
     email: String,
     shippingAddress: String,
     billingAddress: String,
   });
   const User = mongoose.model('User', UserSchema);

   const runConsumer = async () => {
     await consumer.connect();
     await consumer.subscribe({ topic: orderTopic, fromBeginning: true });

     await consumer.run({
       eachMessage: async ({ topic, partition, message }) => {
         console.log(`Processing order event: ${message.value.toString()}`);
         // Process order-related user information
       },
     });
   };
   runConsumer();

   app.listen(3003, () => console.log('User Service running on port 3003'));
   ```

---

#### **Step 5: Test the Setup**
1. Start all services:
   ```bash
   node productService.js
   node orderService.js
   node userService.js
   ```

2. Perform actions using `curl` or Postman:
   - Add a product:
     ```bash
     curl -X POST -H "Content-Type: application/json" -d '{"name": "Laptop", "price": 1000, "stock": 10}' http://localhost:3001/product
     ```
   - Place an order:
     ```bash
     curl -X POST -H "Content-Type: application/json" -d '{"productId": "ID", "userId": "USER_ID", "quantity": 1}' http://localhost:3002/order
     ```

---

Let me know if you'd like further details on enhancements, monitoring, or testing Kafka pipelines!
