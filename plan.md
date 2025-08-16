# Detailed Implementation Plan for Cultural Center Web Application

This plan outlines step-by-step changes and file modifications to build a full-stack Next.js app (using API routes) with SQLite (via Prisma) and free/mock alternatives (Stripe test mode, mock email, random JWT secret). The app supports a Library Module, Online Store Module, and User Management with Google OAuth authentication, all with modern, responsive TailwindCSS UI.

---

## 1. Environment Setup

- **Create a new `.env` file** at the project root with:
  - `DATABASE_URL="file:./dev.db"` (SQLite database)
  - `JWT_SECRET="your_random_jwt_secret"` (random string for JWT signing)
  - `STRIPE_SECRET_KEY="your_stripe_test_key"` (test mode key)
  - `GOOGLE_CLIENT_ID="your_google_client_id"` (Google OAuth client ID)
  - `GOOGLE_CLIENT_SECRET="your_google_client_secret"` (Google OAuth client secret)
  - `NEXTAUTH_URL="http://localhost:3000"` (for NextAuth.js)
  - `NEXTAUTH_SECRET="your_nextauth_secret"` (random string for NextAuth.js)
  - (Optional) `EMAIL_SERVICE_API_KEY` if using nodemailer in production or demo (for notifications)

---

## 2. Dependency and Package Configuration

- **Modify `package.json`:**
  - Add dependencies:
    - `"prisma"`, `"@prisma/client"`
    - `"next-auth"` (for Google OAuth authentication)
    - `"@next-auth/prisma-adapter"` (Prisma adapter for NextAuth)
    - `"jsonwebtoken"` (for JWT handling)
    - `"bcrypt"` or `"bcryptjs"` (for password hashing)
    - `"stripe"` (for payment integration)
  - Add Prisma scripts:
    - `"prisma:migrate": "prisma migrate dev --name init"`
    - `"prisma:generate": "prisma generate"`

---

## 3. Database Schema with Prisma

- **Create a new file:** `/prisma/schema.prisma`
  - **Define models:**
    - **User**: Fields include `id`, `name`, `email`, `password` (optional for Google auth), `role` (enum: ADMIN, CUSTOMER), `image`, `emailVerified`, and relation fields.
    - **Account**: NextAuth.js required model for OAuth providers
    - **Session**: NextAuth.js required model for session management
    - **VerificationToken**: NextAuth.js required model for email verification
    - **Book** (Library): Fields for `id`, `title`, `author`, `category`, `ISBN`, `status` (enum: AVAILABLE, BORROWED).
    - **Loan**: Fields for `id`, `userId`, `bookId`, `loanDate`, `returnDate`, `status` (enum: PENDING, RETURNED); relations to both User and Book.
    - **Product** (Store): Fields for `id`, `title`, `author`, `category`, `description`, `price`, and either `stock` (physical) or `downloadLink` (digital).
    - **Order**: Fields for `id`, `userId`, a list of products ordered, `orderDate`, and `status` (e.g., PENDING, COMPLETED).
  - **Run migrations:**
    - Execute: `npx prisma migrate dev`
    - Generate clients: `npx prisma generate`

---

## 4. NextAuth.js Configuration for Google OAuth

- **File:** `/src/app/api/auth/[...nextauth]/route.ts`
  - Configure NextAuth.js with:
    - Google OAuth provider using `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET`
    - Prisma adapter for database integration
    - Custom callbacks for JWT and session handling
    - Role-based access control integration

- **File:** `/src/lib/auth.ts`
  - Export NextAuth configuration
  - Define authentication options and providers
  - Handle user role assignment (default to CUSTOMER, allow admin promotion)

---

## 5. Backend API Endpoints (Next.js API Routes)

### a. Authentication Endpoints (Enhanced with Google OAuth)

- **File:** `/src/app/api/auth/register/route.ts`
  - Implement a POST endpoint for traditional email/password registration:
    - Validate input (name, email, password).
    - Hash the password using bcrypt.
    - Create a new User record in the database via Prisma.
    - Return a JSON success response.

- **File:** `/src/app/api/auth/login/route.ts`
  - Implement a POST endpoint for traditional login:
    - Validate credentials; compare password using bcrypt.
    - Generate a JWT using jsonwebtoken and sign with `JWT_SECRET`.
    - Set the token in an HttpOnly cookie and return user info.
    - Include error handling for invalid credentials.

- **Google OAuth Integration:**
  - Users can sign in with Google through NextAuth.js
  - Automatic account creation for new Google users
  - Session management handled by NextAuth.js

### b. Library Module Endpoints

- **File:** `/src/app/api/library/books/route.ts`
  - Support multiple HTTP methods:
    - **GET:** List all books.
    - **POST:** Add new book (Admin-only; verify session from NextAuth or JWT from cookies).
    - **PUT/PATCH:** Update a book's details.
    - **DELETE:** Remove a book.
  - Validate required fields: title, author, category, ISBN, status.
  - Include try/catch error handling and proper HTTP status codes.

- **File:** `/src/app/api/library/loans/route.ts`
  - Implement endpoints:
    - **GET:** List all loans or filter by user/status.
    - **POST:** Create a new loan record (capture borrower, loan date, expected return date).
    - **PUT/PATCH:** Update loan status (e.g., mark as RETURNED).
  - Include logic to check for overdue loans and trigger mock notifications (e.g., console logging or email integration in production).

### c. Online Store Module Endpoints

- **File:** `/src/app/api/store/products/route.ts`
  - **GET:** Return the catalog of digital and physical books.
  - **POST:** Add new products (Admin-only verification).
  - Each product record includes: title, author, category, description, price, and either stock (physical) or downloadLink (PDF).

- **File:** `/src/app/api/store/orders/route.ts`
  - **POST:** Create a new order.
    - Validate that the user is authenticated (NextAuth session or JWT).
    - Parse order details from the request.
    - For digital (PDF) products, trigger an immediate download session.
    - For physical products, mark the order for later tracking.
  - Return an order confirmation in JSON.

- **File:** `/src/app/api/store/checkout/route.ts`
  - **POST:** Integrate Stripe to create a Checkout Session:
    - Use the Stripe SDK with test secret.
    - Configure the session with order details and payment mode.
    - Return the Stripe session URL for redirection.
  - Handle errors from Stripe and return proper feedback.

---

## 6. Frontend Page Components (Using Next.js App Router)

### a. User Authentication (Enhanced with Google OAuth)

- **Files:**
  - `/src/app/auth/register/page.tsx`
    - Render a user registration form using a modern TailwindCSS design.
    - Include both traditional email/password registration and Google OAuth button.
    - Use the RegisterForm component (to be created) that calls the `/api/auth/register` endpoint.
  - `/src/app/auth/login/page.tsx`
    - Render a login form using Tailwind classes.
    - Include both traditional login form and Google OAuth button.
    - Use NextAuth.js signIn function for Google authentication.
    - Use the LoginForm component that calls the `api/auth/login` endpoint for traditional login.

- **File:** `/src/components/auth/GoogleSignInButton.tsx`
  - Create a reusable Google sign-in button component
  - Use NextAuth.js signIn function with Google provider
  - Modern styling with Google branding colors

### b. Library Module Views

- **File:** `/src/app/library/page.tsx`
  - Display a grid/list of books using a new `BookCard` component (in `/src/components/library/BookCard.tsx`).
  - Include a filter/sort feature and buttons for Admin to add or edit books.
  - Protect admin features with session-based role checking.
  
- **File:** `/src/app/library/[bookId]/page.tsx`
  - Show detailed view of a selected book with its information and loan option.
  - Include an input form to borrow the book (which calls the `/api/library/loans` endpoint) if available.
  - Require authentication (NextAuth session) to borrow books.

### c. Online Store Views

- **File:** `/src/app/store/page.tsx`
  - Render a product catalog grid of books using a new `ProductCard` component (in `/src/components/store/ProductCard.tsx`).
  - Display product summary: title, author, category, price.
  
- **File:** `/src/app/store/[productId]/page.tsx`
  - Show detailed product information.
  - For digital products, render an immediate download button (post-purchase) with a fallback link.
  
- **File:** `/src/app/cart/page.tsx`
  - Render a shopping cart UI with list of items.
  - Allow quantity adjustments, removal, and a checkout button that calls the `/api/store/checkout` endpoint.
  - Require authentication before checkout.

### d. Administration and User Profile

- **File:** `/src/app/admin/page.tsx`
  - Create an Admin Dashboard with tabs for managing Library (books, loans) and Store (products, orders).
  - Use a custom layout with notifications for overdue books and pending orders, using a `Notification` component.
  - Protect with admin role checking using NextAuth session.
  
- **File:** `/src/app/profile/page.tsx`
  - Render the user's profile.
  - Display the history of loans from the Library and purchase history from the Store.
  - Show user information from Google OAuth or traditional registration.
  - Design for clarity with a modern, responsive layout.

---

## 7. UI Components and Styling

- **Authentication Components:**
  - **Files:** `/src/components/auth/RegisterForm.tsx` and `/src/components/auth/LoginForm.tsx`
    - Build forms using TailwindCSS for spacing, fonts, and colors.
    - Implement error display messages directly below form fields.
    - Include Google OAuth integration alongside traditional forms.
  
  - **File:** `/src/components/auth/GoogleSignInButton.tsx`
    - Modern Google sign-in button with proper branding
    - Handle NextAuth.js signIn function calls
    - Loading states and error handling

- **Navigation and Layout:**
  - **File:** `/src/components/layout/Navbar.tsx`
    - Include user authentication status
    - Show user avatar from Google OAuth or default avatar
    - Sign out functionality using NextAuth.js signOut
    - Role-based navigation items (admin links for admins)

- **Library Components:**
  - **File:** `/src/components/library/BookCard.tsx`
    - A clean card layout displaying book title, author, category, ISBN, and status.
    - Use typography, color contrasts, and spacing in line with a modern UI.
  
- **Store Components:**
  - **File:** `/src/components/store/ProductCard.tsx`
    - Card shows product image (if required, use a placeholder image with:  
      `<img src="https://placehold.co/400x300?text=Detailed+book+cover+image+for+online+store" alt="Detailed book cover image for online store" onerror="this.onerror=null;this.src='fallback.jpg'" />`), title, author, price.
    - Clean layout without external icons.
  
- **General UI:**
  - **File:** `/src/components/ui/Notification.tsx`
    - Use for alerts such as upcoming or overdue loans, styled with modern color backgrounds and clear typography.

---

## 8. Session Management and Protection

- **File:** `/src/lib/auth-utils.ts`
  - Helper functions for session validation
  - Role-based access control utilities
  - Server-side session checking functions

- **File:** `/src/middleware.ts`
  - NextAuth.js middleware for protecting routes
  - Redirect unauthenticated users to login
  - Role-based route protection

- **Protected Routes:**
  - `/admin/*` - Admin only
  - `/profile` - Authenticated users only
  - `/cart` - Authenticated users only
  - Library borrowing - Authenticated users only

---

## 9. Utility Functions and Error Handling

- **File:** `/src/lib/utils.ts`
  - Add helper functions:
    - `generateJWT(payload: object): string` (using jsonwebtoken)
    - `verifyJWT(token: string)` to extract payload.
    - Password hashing and comparison functions using bcrypt.
    - Session validation helpers for NextAuth.js
  - Create a general error response helper function to standardize API responses.

- **Error Handling in API Routes:**
  - Wrap database calls and payment integration code in try/catch blocks.
  - Return HTTP codes (400 for bad request, 401 for unauthorized, 500 for server errors) with descriptive JSON messages.
  - Handle NextAuth.js session validation errors.

---

## 10. Google OAuth Setup Instructions

- **Google Cloud Console Setup:**
  - Create a new project or use existing one
  - Enable Google+ API
  - Create OAuth 2.0 credentials
  - Add authorized redirect URIs: `http://localhost:3000/api/auth/callback/google`
  - Copy Client ID and Client Secret to environment variables

- **Development vs Production:**
  - Use different Google OAuth apps for development and production
  - Update redirect URIs accordingly
  - Ensure proper domain verification for production

---

## 11. Integration & Testing

- **Local Testing:**
  - Start the development server with `npm run dev` or `next dev`.
  - Test Google OAuth flow in browser
  - Use curl commands to test endpoints. For example:
    - Registration:
      ```bash
      curl -X POST http://localhost:3000/api/auth/register \
      -H "Content-Type: application/json" \
      -d '{"name": "John Doe", "email": "john@example.com", "password": "securePass123"}'
      ```
    - Login:
      ```bash
      curl -X POST http://localhost:3000/api/auth/login \
      -H "Content-Type: application/json" \
      -d '{"email": "john@example.com", "password": "securePass123"}'
      ```
    - Similar tests for book CRUD, loan creation, product checkout, etc.
- **Stripe Checkout Test:**
  - Validate the `/api/store/checkout` endpoint returns a session URL by invoking a curl test command and checking the response.
- **Google OAuth Test:**
  - Test complete Google sign-in flow
  - Verify user creation in database
  - Test session persistence and role assignment

---

## 12. Final Considerations

- Implement role-based access control by validating NextAuth.js sessions and JWT tokens in protected endpoints (e.g., only admins can modify books/products).
- Ensure all forms in the UI provide feedback on success/error with modern styling.
- Handle both Google OAuth users and traditional email/password users seamlessly.
- Use proper console logging during development for debugging.
- Document the API endpoints briefly in the README (internal documentation only).
- Implement proper error boundaries for React components.
- Add loading states for all authentication flows.

---

### Summary

- Updated environment files and package.json with dependencies for Prisma, NextAuth.js, JWT, bcrypt, and Stripe.
- Created a Prisma schema (`/prisma/schema.prisma`) with models for users, books, loans, products, orders, and NextAuth.js required models, using SQLite.
- Configured NextAuth.js with Google OAuth provider for easy authentication.
- Built Next.js API routes for authentication, library CRUD, and store order processing with Stripe integration.
- Enhanced authentication with both traditional email/password and Google OAuth options.
- Developed modern UI pages for registration, login, library catalog, online store, cart, admin dashboard, and user profile using TailwindCSS.
- Added dedicated components for forms, cards, notifications, and Google authentication to ensure a consistent user experience.
- Implemented comprehensive session management and route protection.
- Implemented extensive error handling and utility functions in `/src/lib/utils.ts`.
- Planned API testing via curl commands and browser testing for OAuth flows to validate endpoint behavior.

After the plan approval, I will breakdown the plan into logical steps and create a tracker (TODO.md) to track the execution of steps in the plan. I will overwrite this file every time to update the completed steps.
