# 🚀 Vue.js + CouchDB + PouchDB – Crash Project  

This project is a **hands-on learning experience** to understand the basics of **NoSQL databases** using **Vue.js** and **CouchDB**, with **PouchDB** handling local data storage and synchronization.  

## 🎯 Learning Goals  

This project was done **before fully learning Vue.js** on purpose. The idea was to **learn by doing**, figure things out along the way, and solve real development problems instead of just following a tutorial.  

By diving straight in, the project helps to:  

- **Understand NoSQL concepts** using CouchDB and PouchDB.  
- **Learn Vue.js through practice**, even without prior experience.  
- **Become more independent in solving coding challenges.**  
- **Explore how offline-first and distributed databases work.**  

## 🔧 Technologies Used  

- **Vue.js** – JavaScript framework for building the UI.  
- **CouchDB** – A NoSQL database designed for syncing and scaling.  
- **PouchDB** – A browser-based NoSQL database that syncs with CouchDB.  

The app runs at `http://localhost:5173/` (or another port based on your Vue settings).  

## 🛠 CRUD Features  

The app includes basic **CRUD operations**:  

- ✅ **Create** – Add a new document (via a Vue.js form).  
- 🔄 **Read** – Display a list of stored documents.  
- ✏️ **Update** – Edit an existing document.  
- ❌ **Delete** – Remove a document from the database.  

## 🔄 Replication & Syncing  

One key feature is **offline support**. Thanks to **PouchDB**, any changes made while offline are saved locally and automatically **synced with CouchDB** once the internet is available again.  

### 🏗 How It Works  

1. **Local Changes** – When a user adds, edits, or deletes a document, the data is stored in **PouchDB**.  
2. **Sync with CouchDB** – Once back online, PouchDB syncs the local changes with CouchDB.  
3. **Two-Way Sync** – Changes from CouchDB (like updates from another user) are also pulled into PouchDB, keeping everything up to date.  


This makes the app **more reliable**, allowing users to work even when offline, with automatic updates when they reconnect. 🚀
