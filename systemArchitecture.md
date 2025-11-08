#  System Architecture – Price Checker

## 🧩 Overview
The Price Checker system is built to collect, store, and display up-to-date product prices from multiple sources — primarily user submissions — with timestamped verification and geolocation tagging.

![Price Checker Wireframe](https://whimsical.com/AGHEWFjvtJbacVrUTELdxX)
---

## 🖥️ Frontend
**Stack:** React / Next.js  
**Purpose:**  
- Provide an intuitive interface for users to search, view, and compare prices.  
- Allow users to submit price updates and filter by location or time.  
**Key Components:**  
- Search & Filter Module  
- Product Comparison View  
- Add/Update Price Form  
- Authentication (optional in later version)

---

## 🧠 Backend
**Stack:** Node.js (Express)  
**Purpose:**  
- Handle API requests and manage communication between frontend and database.  
- Verify, store, and serve price data with timestamps and location.  
**Modules:**  
- Product Management API  
- Price Update & History API  
- Location Service (integration with Google Maps API)

---

## 🗃️ Database
**Type:** MongoDB (NoSQL)  
**Collections:**  
- `users` – user details and optional profile info  
- `products` – product names, categories  
- `prices` – store name, price, timestamp, location, user ID  

---

## 🔄 Data Flow
1. User searches for a product → Frontend sends request to Backend API.  
2. Backend fetches latest price data from `prices` collection.  
3. User can submit new price → Backend validates → stores in DB with timestamp.  
4. API sends updated results back to frontend for display.

---

## 🧱 Communication
- **Frontend ↔ Backend:** RESTful API (JSON)  
- **Backend ↔ Database:** Mongoose ODM  
- **Optional External Services:**  
  - Google Maps API for location lookup  
  - Firebase / Auth0 for authentication (future versions)

---

## 🚀 Feasibility
The system uses proven, lightweight web technologies that scale easily with user growth.  
MongoDB’s flexible schema supports frequent updates, while React ensures fast UI performance and real-time refresh capabilities.
