# Marcelo_test

### Introduction

Pour tester vos routes d'API avec Jest et Sequelize, nous utiliserons Jest pour exécuter les tests et `supertest` pour simuler des requêtes HTTP vers vos routes. Jest est un framework de test JavaScript qui permet d'écrire des tests unitaires et d'intégration. `supertest` est une bibliothèque qui facilite les tests d'API en simulant des requêtes HTTP.

### Installation de Jest et Supertest

1. **Installer Jest et Supertest**

   Utilisez npm pour installer Jest et `supertest` en tant que dépendances de développement :

   ```bash
   npm install --save-dev jest supertest
   ```

2. **Configurer le script de test**

   Ajoutez un script de test dans votre fichier `package.json` :

   ```json
   "scripts": {
     "test": "jest"
   }
   ```

### Configurer l'environnement de test

1. **Configurer une base de données de test**

   Créez un fichier `db.js` pour gérer la connexion à la base de données. Nous allons utiliser SQLite en mémoire pour les tests.

   ```javascript
   // db.js
   const { Sequelize } = require('sequelize');

   const env = process.env.NODE_ENV || 'development';
   let sequelize;

   if (env === 'test') {
     sequelize = new Sequelize('sqlite::memory:', {
       logging: false,
     });
   } else {
     sequelize = new Sequelize('smartspend_db', 'root', '', {
       host: 'localhost',
       dialect: 'mysql',
     });
   }

   module.exports = sequelize;
   ```

2. **Configurer Jest**

   Ajoutez un fichier de configuration pour Jest, `jest.config.js`, à la racine de votre projet.

   ```javascript
   // jest.config.js
   module.exports = {
     testEnvironment: 'node',
     setupFilesAfterEnv: ['./jest.setup.js'],
   };
   ```

3. **Configurer le fichier de configuration Jest pour initialiser la base de données**

   Créez un fichier `jest.setup.js` pour synchroniser les modèles et ajouter des données de test avant chaque suite de tests.

   ```javascript
   // jest.setup.js
   const sequelize = require('./db');
   const { User } = require('./models');  // Assurez-vous que c'est le bon chemin vers vos modèles
   const bcrypt = require('bcrypt');

   beforeAll(async () => {
     await sequelize.sync({ force: true });

     // Ajouter des données de test
     const hashedPassword = await bcrypt.hash('password123', 10);
     await User.bulkCreate([
       { name: 'John', surname: 'Doe', email: 'john@example.com', password: hashedPassword },
       { name: 'Jane', surname: 'Doe', email: 'jane@example.com', password: hashedPassword },
       { name: 'Bob', surname: 'Smith', email: 'bob@example.com', password: hashedPassword },
     ]);
   });

   afterAll(async () => {
     await sequelize.close();
   });
   ```

### Écrire les tests

Créez un fichier `user.test.js` dans le dossier `tests` et ajoutez vos tests.

```javascript
// tests/user.test.js

const request = require('supertest');
const app = require('../app');  // Assurez-vous que c'est le bon chemin vers votre application Express
const { User } = require('../models');  // Assurez-vous que c'est le bon chemin vers votre modèle

describe('User API', () => {
  it('should create a new user', async () => {
    const res = await request(app)
      .post('/users')
      .send({ name: 'Alice', surname: 'Johnson', email: 'alice@example.com', password: 'password123' });
    
    expect(res.status).toBe(201);
    expect(res.body).toHaveProperty('name', 'Alice');
    expect(res.body).toHaveProperty('surname', 'Johnson');
    expect(res.body).toHaveProperty('email', 'alice@example.com');
  });

  it('should check if username is unique', async () => {
    const res = await request(app).get('/users/checkUsernameUnique/Smith');
    
    expect(res.status).toBe(200);
    expect(res.body).toHaveProperty('exists', true);
  });

  it('should retrieve all users', async () => {
    const res = await request(app).get('/users');
    
    expect(res.status).toBe(200);
    expect(Array.isArray(res.body)).toBe(true);
    expect(res.body.length).toBeGreaterThanOrEqual(3);  // assuming at least the 3 seeded users
  });

  it('should retrieve a user by ID', async () => {
    const user = await User.findOne({ where: { email: 'john@example.com' } });
    const res = await request(app).get(`/users/${user.id}`);
    
    expect(res.status).toBe(200);
    expect(res.body).toHaveProperty('name', 'John');
  });

  it('should update a user', async () => {
    const user = await User.findOne({ where: { email: 'jane@example.com' } });
    const res = await request(app)
      .put(`/users/${user.id}`)
      .send({ name: 'Jane Updated' });
    
    expect(res.status).toBe(200);
    expect(res.body).toHaveProperty('name', 'Jane Updated');
  });

  it('should delete a user', async () => {
    const user = await User.findOne({ where: { email: 'bob@example.com' } });
    const res = await request(app).delete(`/users/${user.id}`);
    
    expect(res.status).toBe(200);
    expect(res.body).toHaveProperty('message', 'User deleted');
  });
});
```

### Exécuter les tests

Exécutez les tests avec la commande suivante :

```bash
npm test
```

Cette configuration garantit que chaque fois que vous exécutez vos tests, la base de données de test est initialisée avec les données de test spécifiées. Cela vous permet de vérifier les comportements attendus de vos routes et modèles dans un état de base de données prédéfini.
