# 🚀 Vue.js + CouchDB + PouchDB – Crash Project  

This project is a **hands-on experiment** to learn the fundamentals of **NoSQL databases** using **Vue.js** and **CouchDB**, with **PouchDB** as a local synchronization layer.  

## 🎯 Educational and Personal Objectives  

This project was **deliberately undertaken before mastering Vue.js**, following a learning-by-doing approach. The goal was to **stimulate resourcefulness**, face real-world development challenges, and learn how to research, experiment, and adapt rather than just following a theoretical approach.  

By getting directly “hands-on,” this project aims to:  

- **Learn NoSQL concepts** through CouchDB and synchronization with PouchDB.  
- **Experiment with Vue.js in real-world conditions**, without prior academic knowledge.  
- **Develop autonomy in solving technical problems.**  
- **Understand how distributed and offline-first databases work.**  

## 🔧 Technologies Used  

- **Vue.js** – JavaScript framework for the user interface.  
- **CouchDB** – Document-oriented NoSQL database, designed for replication and resilience.  
- **PouchDB** – Client-side NoSQL database enabling synchronization with CouchDB.  

The application will be accessible at `http://localhost:5173/` (or another port based on your Vue configuration).  

## 🛠 CRUD Features  

The application allows performing the following **CRUD operations**:  

- ✅ **Create**: Add a document to the database (Vue.js form).  
- 🔄 **Read**: Display a list of stored documents in CouchDB.  
- ✏️ **Update**: Modify an existing document through the interface.  
- ❌ **Delete**: Remove a document from the database.  

Thanks to **PouchDB**, local changes are automatically **synchronized** with CouchDB whenever an internet connection is available.
