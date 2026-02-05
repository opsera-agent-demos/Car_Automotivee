# Car Automotive - Full-Stack Web Application

A modern, production-ready full-stack car automotive platform built with React, Node.js, Express, and MongoDB.

---

## 🚀 Deployment Landscape

```
╔══════════════════════════════════════════════════════════════════════════════════════════════════════╗
║                                                                                                      ║
║     ██████╗  █████╗ ██████╗       █████╗ ██╗   ██╗████████╗ ██████╗ ███╗   ███╗ ██████╗ ████████╗   ║
║    ██╔════╝ ██╔══██╗██╔══██╗     ██╔══██╗██║   ██║╚══██╔══╝██╔═══██╗████╗ ████║██╔═══██╗╚══██╔══╝   ║
║    ██║      ███████║██████╔╝     ███████║██║   ██║   ██║   ██║   ██║██╔████╔██║██║   ██║   ██║      ║
║    ██║      ██╔══██║██╔══██╗     ██╔══██║██║   ██║   ██║   ██║   ██║██║╚██╔╝██║██║   ██║   ██║      ║
║    ╚██████╗ ██║  ██║██║  ██║     ██║  ██║╚██████╔╝   ██║   ╚██████╔╝██║ ╚═╝ ██║╚██████╔╝   ██║      ║
║     ╚═════╝ ╚═╝  ╚═╝╚═╝  ╚═╝     ╚═╝  ╚═╝ ╚═════╝    ╚═╝    ╚═════╝ ╚═╝     ╚═╝ ╚═════╝    ╚═╝      ║
║                                                                                                      ║
║                              DEPLOYMENT LANDSCAPE                                                    ║
║                                                                                                      ║
╠══════════════════════════════════════════════════════════════════════════════════════════════════════╣
║  Application: car-automotive       Tenant: opsera           Region: us-west-2                        ║
║  Generated: 2026-02-05             Powered by: Opsera Code-to-Cloud Enterprise v0.917                ║
╚══════════════════════════════════════════════════════════════════════════════════════════════════════╝
```

### Environment Status

| Environment | URL | Status | Strategy | Replicas |
|-------------|-----|--------|----------|----------|
| **DEV** | [opsera-car-automotive-dev.agent.opsera.dev](https://opsera-car-automotive-dev.agent.opsera.dev) | ✅ Active | Rolling | 2 |

### Infrastructure Overview

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              ARCHITECTURE DIAGRAM                                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│    ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                      │
│    │   GitHub     │────▶│  GitHub      │────▶│   AWS ECR    │                      │
│    │   (Source)   │     │  Actions     │     │  (Registry)  │                      │
│    └──────────────┘     └──────────────┘     └──────────────┘                      │
│           │                    │                    │                               │
│           │                    │                    ▼                               │
│           │              ┌─────┴─────┐     ┌──────────────┐                        │
│           │              │           │     │   ArgoCD     │                        │
│           │              │  Gitleaks │     │   (GitOps)   │                        │
│           │              │  + Grype  │     │  Hub Cluster │                        │
│           │              │           │     └──────────────┘                        │
│           │              └───────────┘            │                                │
│           │                                       │                                │
│           ▼                                       ▼                                │
│    ┌──────────────┐                      ┌──────────────┐                         │
│    │ Kustomize    │◀─────────────────────│    EKS       │                         │
│    │ Manifests    │                      │ Spoke Cluster│                         │
│    └──────────────┘                      └──────────────┘                         │
│                                                 │                                  │
│                                    ┌────────────┼────────────┐                    │
│                                    ▼            ▼            ▼                    │
│                             ┌──────────┐ ┌──────────┐ ┌──────────┐               │
│                             │ Frontend │ │ Backend  │ │ Ingress  │               │
│                             │  (Nginx) │ │ (Node.js)│ │ (nginx)  │               │
│                             └──────────┘ └──────────┘ └──────────┘               │
│                                                                                   │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### CI/CD Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                               CI/CD PIPELINE FLOW                                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐         │
│   │  Push   │───▶│ Gitleaks│───▶│  Build  │───▶│  Grype  │───▶│  Push   │         │
│   │  Code   │    │  Scan   │    │  Image  │    │  Scan   │    │  ECR    │         │
│   └─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘         │
│                                                                     │              │
│                                                                     ▼              │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐         │
│   │  Live   │◀───│ ArgoCD  │◀───│  Apply  │◀───│  Commit │◀───│ Update  │         │
│   │   App   │    │  Sync   │    │Manifests│    │  Push   │    │  Tags   │         │
│   └─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘         │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Quick Links

| Resource | Link |
|----------|------|
| **Application** | [https://opsera-car-automotive-dev.agent.opsera.dev](https://opsera-car-automotive-dev.agent.opsera.dev) |
| **GitHub Repo** | [opsera-agent-demos/Car_Automotivee](https://github.com/opsera-agent-demos/Car_Automotivee) |
| **ArgoCD** | [argocd-usw2.agent.opsera.dev](https://argocd-usw2.agent.opsera.dev) |
| **ECR Frontend** | `792373136340.dkr.ecr.us-west-2.amazonaws.com/opsera/car-automotive-frontend` |
| **ECR Backend** | `792373136340.dkr.ecr.us-west-2.amazonaws.com/opsera/car-automotive-backend` |

### Recent Deployments

| Date | Commit | Environment | Status |
|------|--------|-------------|--------|
| 2026-02-05 | `4f42ace` | DEV | ✅ Success |
| 2026-02-05 | `5738628` | DEV | ✅ Success |
| 2026-02-05 | `402b48a` | DEV | ✅ Success |
| 2026-02-05 | `3487d14` | DEV | ✅ Success |

### GitHub Actions Workflows

| Workflow | Purpose | Trigger |
|----------|---------|---------|
| [CI Build & Push (DEV)](../../actions/workflows/ci-build-push-car-automotive-dev.yaml) | Build, scan, push images, deploy | Push to main/car-automotive |
| [Bootstrap Infrastructure](../../actions/workflows/bootstrap-car-automotive.yaml) | Create ECR repos, namespaces, ArgoCD apps | Manual |
| [Verify Pods](../../actions/workflows/verify-pods-car-automotive.yaml) | Debug: Check pod status | Manual |
| [Test URLs](../../actions/workflows/test-urls-car-automotive.yaml) | Debug: Test application URLs | Manual |
| [Diagnostics](../../actions/workflows/diagnostics-car-automotive.yaml) | Full pipeline diagnostics | Manual |
| [ArgoCD Sync](../../actions/workflows/argocd-sync-car-automotive.yaml) | Force ArgoCD sync | Manual |

---

## 📁 Opsera Infrastructure Files

```
.opsera-car-automotive/
├── argocd/
│   └── dev/
│       └── application.yaml          # ArgoCD Application manifest
├── k8s/
│   ├── base/
│   │   ├── kustomization.yaml        # Base Kustomize config
│   │   ├── frontend-deployment.yaml  # Frontend Deployment
│   │   ├── frontend-service.yaml     # Frontend Service
│   │   ├── backend-deployment.yaml   # Backend Deployment
│   │   ├── backend-service.yaml      # Backend Service
│   │   └── ingress.yaml              # Ingress configuration
│   └── overlays/
│       └── dev/
│           ├── kustomization.yaml    # Dev overlay with image tags
│           └── namespace.yaml        # Namespace definition
├── Dockerfiles/
│   ├── Dockerfile.frontend           # React/Nginx Dockerfile
│   └── Dockerfile.backend            # Node.js Dockerfile
├── nginx.conf                        # Nginx configuration (port 8080)
├── SKILL-COMPLETE-v0.915.md          # Session learnings & fixes
└── LEARNINGS-2026-02-05.md           # Deployment learnings
```

---

## 🔧 Deployment Configuration

### Kubernetes Resources

| Resource | Name | Namespace |
|----------|------|-----------|
| Namespace | `opsera-car-automotive-dev` | - |
| Deployment | `car-automotive-frontend` | `opsera-car-automotive-dev` |
| Deployment | `car-automotive-backend` | `opsera-car-automotive-dev` |
| Service | `car-automotive-frontend` | `opsera-car-automotive-dev` |
| Service | `car-automotive-backend` | `opsera-car-automotive-dev` |
| Ingress | `car-automotive` | `opsera-car-automotive-dev` |

### Container Configuration

| Component | Image | Port | UID | Health Check |
|-----------|-------|------|-----|--------------|
| Frontend | `nginx-unprivileged:alpine` | 8080 | 101 | `GET /` |
| Backend | `node:20-alpine` | 8080 | 1001 | `GET /api/health` |

### ArgoCD Configuration

```yaml
Application: car-automotive-dev
Repository: https://github.com/opsera-agent-demos/Car_Automotivee.git
Branch: car-automotive
Path: .opsera-car-automotive/k8s/overlays/dev
Destination: opsera-usw2-np (spoke cluster)
Sync Policy: Automated (prune + selfHeal)
```

---

## 🚀 Features

### User Features
- User authentication (Register, Login, Logout)
- Profile management
- Browse cars by brand, price, fuel type, transmission, body type
- Car detail pages with image gallery, specifications, and features
- Book test drives (date & location)
- Add cars to wishlist
- View booking history
- Search and filter cars
- Responsive design with dark mode support

### Admin Features
- Secure admin dashboard
- Add/Edit/Delete cars
- Upload multiple car images (Cloudinary integration)
- Manage test drive bookings
- Manage users and roles
- Mark cars as featured/out of stock
- Analytics dashboard with statistics

## 🛠️ Tech Stack

### Frontend
- React 18
- Vite
- Tailwind CSS
- Redux Toolkit
- React Router
- Axios
- React Hot Toast

### Backend
- Node.js
- Express.js
- MongoDB with Mongoose
- JWT Authentication (Access + Refresh Tokens)
- Cloudinary (Image Storage)
- Express Validator

## 📁 Project Structure

```
car-automotive/
├── backend/
│   ├── config/
│   │   └── cloudinary.config.js
│   ├── controllers/
│   │   ├── admin.controller.js
│   │   ├── auth.controller.js
│   │   ├── booking.controller.js
│   │   ├── car.controller.js
│   │   └── user.controller.js
│   ├── middleware/
│   │   ├── auth.middleware.js
│   │   └── errorHandler.js
│   ├── models/
│   │   ├── Booking.model.js
│   │   ├── Car.model.js
│   │   └── User.model.js
│   ├── routes/
│   │   ├── admin.routes.js
│   │   ├── auth.routes.js
│   │   ├── booking.routes.js
│   │   ├── car.routes.js
│   │   └── user.routes.js
│   ├── utils/
│   │   └── generateToken.js
│   ├── .env.example
│   ├── .gitignore
│   ├── package.json
│   └── server.js
│
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── auth/
│   │   │   ├── cars/
│   │   │   ├── common/
│   │   │   └── layout/
│   │   ├── pages/
│   │   │   ├── admin/
│   │   │   └── ...
│   │   ├── store/
│   │   │   ├── slices/
│   │   │   ├── api.js
│   │   │   └── store.js
│   │   ├── App.jsx
│   │   ├── main.jsx
│   │   └── index.css
│   ├── index.html
│   ├── package.json
│   ├── tailwind.config.js
│   └── vite.config.js
│
├── .opsera-car-automotive/     # Opsera CI/CD Infrastructure
│   ├── argocd/
│   ├── k8s/
│   ├── Dockerfiles/
│   └── nginx.conf
│
├── .github/
│   └── workflows/              # GitHub Actions CI/CD
│
└── README.md
```

## 🚦 Getting Started

### Prerequisites
- Node.js (v16 or higher)
- MongoDB Atlas account (FREE - recommended, no installation needed) OR Local MongoDB
- Cloudinary account (FREE tier available - for image uploads)

### Backend Setup

1. Navigate to backend directory:
```bash
cd backend
```

2. Install dependencies:
```bash
npm install
```

3. Create `.env` file from `.env.example`:
```bash
cp .env.example .env
```

4. Update `.env` with your configuration:
```env
PORT=5000
NODE_ENV=development
# MongoDB Atlas connection string (recommended - no installation needed!)
MONGODB_URI=mongodb+srv://username:password@cluster0.xxxxx.mongodb.net/car-automotive?retryWrites=true&w=majority
# Or for local MongoDB: mongodb://localhost:27017/car-automotive
JWT_SECRET=your-super-secret-jwt-key
JWT_REFRESH_SECRET=your-super-secret-refresh-jwt-key
JWT_EXPIRE=7d
JWT_REFRESH_EXPIRE=30d
CLOUDINARY_CLOUD_NAME=your-cloud-name
CLOUDINARY_API_KEY=your-api-key
CLOUDINARY_API_SECRET=your-api-secret
FRONTEND_URL=http://localhost:5173
```

**📝 Note:** See `SETUP.md` for detailed MongoDB Atlas setup instructions (it's free and takes 5 minutes!)

5. Start the backend server:
```bash
npm run dev
```

The backend will run on `http://localhost:5000`

### Frontend Setup

1. Navigate to frontend directory:
```bash
cd frontend
```

2. Install dependencies:
```bash
npm install
```

3. Start the development server:
```bash
npm run dev
```

The frontend will run on `http://localhost:5173`

## 📝 API Endpoints

### Authentication
- `POST /api/auth/register` - Register new user
- `POST /api/auth/login` - Login user
- `POST /api/auth/logout` - Logout user
- `POST /api/auth/refresh-token` - Refresh access token
- `GET /api/auth/me` - Get current user

### Cars
- `GET /api/cars` - Get all cars (with filters)
- `GET /api/cars/featured` - Get featured cars
- `GET /api/cars/search` - Search cars
- `GET /api/cars/:id` - Get car by ID

### Bookings
- `POST /api/bookings` - Create booking (Protected)
- `GET /api/bookings/my-bookings` - Get user bookings (Protected)
- `GET /api/bookings/:id` - Get booking by ID (Protected)
- `PUT /api/bookings/:id/cancel` - Cancel booking (Protected)
- `PUT /api/bookings/:id/status` - Update booking status (Admin)

### Users
- `GET /api/users/profile` - Get user profile (Protected)
- `PUT /api/users/profile` - Update profile (Protected)
- `GET /api/users/wishlist` - Get wishlist (Protected)
- `POST /api/users/wishlist/:carId` - Add to wishlist (Protected)
- `DELETE /api/users/wishlist/:carId` - Remove from wishlist (Protected)

### Admin
- `GET /api/admin/dashboard` - Get dashboard stats (Admin)
- `GET /api/admin/cars` - Get all cars (Admin)
- `POST /api/admin/cars` - Create car (Admin)
- `PUT /api/admin/cars/:id` - Update car (Admin)
- `DELETE /api/admin/cars/:id` - Delete car (Admin)
- `GET /api/admin/bookings` - Get all bookings (Admin)
- `GET /api/admin/users` - Get all users (Admin)
- `PUT /api/admin/users/:id/role` - Update user role (Admin)
- `DELETE /api/admin/users/:id` - Delete user (Admin)

## 🗄️ Database Schema

### User
- name (String, required)
- email (String, required, unique)
- password (String, required, hashed)
- role (String, enum: ['USER', 'ADMIN'], default: 'USER')
- wishlist (Array of Car IDs)
- createdAt (Date)

### Car
- name (String, required)
- brand (String, required)
- price (Number, required)
- images (Array of {url, publicId})
- fuelType (String, enum)
- transmission (String, enum)
- mileage (Number)
- engine (String)
- features (Array of Strings)
- specifications (Object: bodyType, seatingCapacity, safetyFeatures, color)
- availability (String, enum)
- featured (Boolean)
- description (String)
- createdAt (Date)

### Booking
- userId (ObjectId, ref: User)
- carId (ObjectId, ref: Car)
- date (Date, required)
- location (String, required)
- status (String, enum: ['Pending', 'Confirmed', 'Completed', 'Cancelled'])
- notes (String)
- createdAt (Date)

## 🔐 Authentication

The application uses JWT (JSON Web Tokens) for authentication:
- Access tokens stored in localStorage
- Refresh tokens stored in httpOnly cookies
- Protected routes require valid JWT token
- Admin routes require ADMIN role

## 🎨 Features

- **Responsive Design**: Mobile-first approach with Tailwind CSS
- **Dark Mode**: Built-in dark mode support
- **Image Upload**: Cloudinary integration for car images
- **Search & Filter**: Advanced filtering and search capabilities
- **Pagination**: Efficient data loading with pagination
- **Error Handling**: Centralized error handling
- **Validation**: Input validation on both frontend and backend

## 📦 Production Build

### Backend
```bash
cd backend
npm start
```

### Frontend
```bash
cd frontend
npm run build
npm run preview
```

## 🤝 Contributing

1. Fork the repository
2. Create your feature branch
3. Commit your changes
4. Push to the branch
5. Open a Pull Request

## 📄 License

This project is licensed under the ISC License.

## 👨‍💻 Author

Built with ❤️ for car automotive enthusiasts

---

**Deployment powered by [Opsera Code-to-Cloud Enterprise v0.917](https://opsera.io)**

**Note**: Make sure to set up your environment variables properly before running the application. For production, use secure secrets and enable HTTPS.
