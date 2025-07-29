# EdenBank Frontend Architecture Guide 2025

## Table of Contents
1. [Technology Stack](#technology-stack)
2. [Architecture Overview](#architecture-overview)
3. [Development Environment Setup](#development-environment-setup)
4. [API Integration Strategy](#api-integration-strategy)
5. [Authentication & Security](#authentication--security)
6. [UI/UX Guidelines](#uiux-guidelines)
7. [State Management](#state-management)
8. [Testing Strategy](#testing-strategy)
9. [Performance Optimization](#performance-optimization)
10. [Deployment Strategy](#deployment-strategy)

## Technology Stack

### Core Framework
- **Next.js 14+** - React framework with server-side rendering, API routes, and excellent performance
- **React 18+** - Component-based UI library with hooks and concurrent features
- **TypeScript 5+** - Type safety and better developer experience

### UI Component Library
- **Ant Design 5+** - Enterprise-focused component library with excellent data visualization
  - Proven in financial applications (Ant Financial)
  - Extensive form controls and data tables
  - Built-in internationalization
  - TypeScript support

### State Management
- **Zustand** - Lightweight state management (recommended for simplicity)
- **React Query (TanStack Query)** - Server state management and caching
- **React Hook Form** - Form state management with validation

### Styling
- **Tailwind CSS** - Utility-first CSS framework
- **CSS Modules** - Component-scoped styling
- **Ant Design Theme** - Customizable design tokens

### Development Tools
- **ESLint** - Code linting with TypeScript support
- **Prettier** - Code formatting
- **Husky** - Git hooks for pre-commit checks
- **Jest** - Unit testing
- **React Testing Library** - Component testing
- **Playwright** - E2E testing

### API Integration
- **Axios** - HTTP client with interceptors
- **OpenAPI TypeScript Codegen** - Generate types from Fineract API
- **React Query** - Data fetching and caching

## Architecture Overview

```
edenbank-frontend/
├── src/
│   ├── app/                    # Next.js app directory
│   │   ├── (auth)/            # Auth layout group
│   │   ├── (dashboard)/       # Dashboard layout group
│   │   ├── api/               # API routes
│   │   └── layout.tsx         # Root layout
│   ├── components/            # Reusable components
│   │   ├── common/           # Generic components
│   │   ├── forms/            # Form components
│   │   ├── charts/           # Data visualization
│   │   └── layouts/          # Layout components
│   ├── features/              # Feature-based modules
│   │   ├── auth/             # Authentication
│   │   ├── clients/          # Client management
│   │   ├── loans/            # Loan management
│   │   ├── savings/          # Savings accounts
│   │   ├── accounting/       # GL accounting
│   │   └── reports/          # Reporting
│   ├── lib/                   # Core libraries
│   │   ├── api/              # API client setup
│   │   ├── hooks/            # Custom hooks
│   │   ├── utils/            # Utility functions
│   │   └── types/            # TypeScript types
│   ├── services/              # API service layer
│   └── styles/                # Global styles
├── public/                    # Static assets
├── tests/                     # Test files
└── config/                    # Configuration files
```

## Development Environment Setup

### Prerequisites
```bash
# Required versions
Node.js: 20+
npm: 10+ or yarn: 1.22+ or pnpm: 8+
```

### Initial Setup
```bash
# Create Next.js project with TypeScript
npx create-next-app@latest edenbank-frontend --typescript --tailwind --app

# Install dependencies
npm install @ant-design/nextjs-registry antd @tanstack/react-query zustand axios
npm install react-hook-form @hookform/resolvers zod
npm install dayjs numeral lodash-es
npm install recharts @ant-design/charts

# Dev dependencies
npm install -D @types/lodash-es @types/numeral
npm install -D @tanstack/eslint-plugin-query
npm install -D openapi-typescript-codegen
```

## API Integration Strategy

### API Client Setup
```typescript
// lib/api/client.ts
import axios, { AxiosInstance } from 'axios';

const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL || 'https://localhost:8443/fineract-provider/api/v1';

export const apiClient: AxiosInstance = axios.create({
  baseURL: API_BASE_URL,
  headers: {
    'Content-Type': 'application/json',
    'Fineract-Platform-TenantId': 'default',
  },
  httpsAgent: new https.Agent({
    rejectUnauthorized: process.env.NODE_ENV === 'production'
  })
});

// Request interceptor for auth
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('authToken');
  if (token) {
    config.headers.Authorization = `Basic ${token}`;
  }
  return config;
});

// Response interceptor for error handling
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      // Handle token refresh or redirect to login
    }
    return Promise.reject(error);
  }
);
```

### Type Generation from OpenAPI
```bash
# Generate TypeScript types from Fineract OpenAPI spec
npx openapi-typescript-codegen --input https://localhost:8443/fineract-provider/swagger-ui/openapi.json --output ./src/lib/api/generated
```

## Authentication & Security

### Authentication Flow
1. **Basic Authentication** (initial implementation)
   - Base64 encode username:password
   - Store in secure httpOnly cookie
   - Send as Authorization header

2. **Two-Factor Authentication** (if enabled)
   - Request OTP after initial auth
   - Store TFA token separately
   - Include Fineract-Platform-TFA-Token header

### Security Best Practices
- Always use HTTPS (Fineract enforces this)
- Implement CSRF protection
- Use Content Security Policy headers
- Sanitize all user inputs
- Implement rate limiting
- Regular security audits

### Auth Implementation
```typescript
// features/auth/hooks/useAuth.ts
import { create } from 'zustand';

interface AuthState {
  user: User | null;
  isAuthenticated: boolean;
  login: (credentials: LoginCredentials) => Promise<void>;
  logout: () => void;
  checkAuth: () => Promise<void>;
}

export const useAuth = create<AuthState>((set) => ({
  user: null,
  isAuthenticated: false,
  login: async (credentials) => {
    // Implement login logic
  },
  logout: () => {
    // Clear auth tokens
    localStorage.removeItem('authToken');
    set({ user: null, isAuthenticated: false });
  },
  checkAuth: async () => {
    // Verify auth status
  }
}));
```

## UI/UX Guidelines

### Design Principles
1. **Clarity** - Clear navigation and intuitive workflows
2. **Efficiency** - Minimize clicks for common tasks
3. **Trust** - Professional appearance for financial data
4. **Accessibility** - WCAG 2.1 AA compliance
5. **Responsiveness** - Mobile-first design

### Component Guidelines
- Use Ant Design components as base
- Customize theme for brand consistency
- Create composite components for complex workflows
- Implement loading states for all async operations
- Provide clear error messages and recovery paths

### Theme Configuration
```typescript
// app/providers/ThemeProvider.tsx
import { ConfigProvider } from 'antd';

const theme = {
  token: {
    colorPrimary: '#1890ff',
    borderRadius: 4,
    fontFamily: 'Inter, system-ui, sans-serif',
  },
  components: {
    Button: {
      controlHeight: 40,
    },
    Input: {
      controlHeight: 40,
    },
  },
};
```

## State Management

### Client State (Zustand)
- User preferences
- UI state (modals, sidebars)
- Form draft data

### Server State (React Query)
- API data caching
- Background refetching
- Optimistic updates
- Infinite queries for pagination

### Example Query Hook
```typescript
// features/clients/hooks/useClients.ts
import { useQuery } from '@tanstack/react-query';
import { clientsService } from '@/services/clients';

export const useClients = (params?: ClientSearchParams) => {
  return useQuery({
    queryKey: ['clients', params],
    queryFn: () => clientsService.getClients(params),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
};
```

## Testing Strategy

### Unit Tests
- Test utilities and helper functions
- Test custom hooks with React Hooks Testing Library
- Test API service methods

### Component Tests
- Test component rendering
- Test user interactions
- Test form validations
- Mock API responses

### E2E Tests
- Critical user journeys
- Authentication flow
- Client onboarding
- Loan application process
- Payment workflows

## Performance Optimization

### Code Splitting
- Route-based splitting (automatic with Next.js)
- Component lazy loading for heavy components
- Dynamic imports for large libraries

### Data Optimization
- Implement pagination for large datasets
- Use virtual scrolling for long lists
- Optimize API payload sizes
- Implement field-level permissions

### Caching Strategy
- Browser caching for static assets
- React Query caching for API data
- Service Worker for offline support
- CDN for static resources

## Deployment Strategy

### Build Process
```bash
# Production build
npm run build

# Type checking
npm run type-check

# Linting
npm run lint

# Tests
npm run test
npm run test:e2e
```

### Environment Configuration
```env
# .env.production
NEXT_PUBLIC_API_URL=https://api.edenbank.com/fineract-provider/api/v1
NEXT_PUBLIC_TENANT_ID=default
```

### CI/CD Pipeline
1. Run tests and linting
2. Build Docker image
3. Deploy to staging
4. Run E2E tests
5. Deploy to production
6. Monitor performance

### Monitoring
- Error tracking (Sentry)
- Performance monitoring
- User analytics
- API response times

## Complete Feature List (Based on Fineract Backend)

### Core Banking Features
1. **Client Management**
   - Client registration (individual/corporate)
   - Client search and filtering
   - Client documents and images
   - Client identifiers (ID cards, passports)
   - Family members management
   - Client address management
   - Client charges

2. **Savings Accounts**
   - Regular savings accounts
   - Fixed deposit accounts
   - Recurring deposit accounts
   - Account transactions (deposit, withdrawal, transfer)
   - Interest calculation and posting
   - Account charges and fees
   - On-hold funds management
   - Dormancy tracking

3. **Loan Management**
   - Loan applications and approval
   - Loan disbursement
   - Repayment schedules
   - Loan transactions (repayments, prepayments)
   - Loan charges and penalties
   - Guarantors management
   - Collateral management
   - Loan restructuring/rescheduling
   - Write-offs and recoveries
   - Post-dated checks

4. **Share Accounts**
   - Share products configuration
   - Share account management
   - Dividend calculations and distributions

5. **Groups & Centers**
   - Group formation and management
   - Center management
   - Collection sheets
   - Group loans
   - Meeting calendar

### Financial Management
6. **Accounting (GL)**
   - Chart of accounts
   - Journal entries
   - Account reconciliation
   - Trial balance
   - Financial reports
   - Provisioning

7. **Teller/Cashier Operations**
   - Cash management
   - Teller allocation
   - Cashier transactions
   - Vault management

8. **Fund Management**
   - Fund sources
   - Fund allocation

### Product Configuration
9. **Product Management**
   - Savings products
   - Loan products
   - Fixed deposit products
   - Recurring deposit products
   - Share products
   - Product mix rules
   - Interest rate charts

10. **Charges & Fees**
    - Charge definitions
    - Fee calculations
    - Tax components and groups

### Administrative Features
11. **Organization**
    - Office hierarchy
    - Staff management
    - Working days configuration
    - Holidays management
    - Currency configuration

12. **User Administration**
    - User management
    - Roles and permissions
    - Password policies
    - Two-factor authentication

13. **System Configuration**
    - Global configurations
    - Code values
    - External services
    - Hooks (webhooks)
    - Data tables

### Self-Service Features
14. **Self-Service Portal**
    - Client self-registration
    - Account access
    - Loan applications
    - Transaction history
    - Beneficiary management
    - Device registration

### Reporting & Analytics
15. **Reports**
    - Client reports
    - Loan reports
    - Savings reports
    - Financial reports
    - Regulatory reports
    - Custom reports (Pentaho)
    - Dashboard KPIs

16. **Search & Queries**
    - Global search
    - Advanced filters
    - Ad-hoc queries

### Communication
17. **Notifications**
    - SMS integration
    - Email notifications
    - In-app notifications
    - Campaign management

18. **Templates**
    - Document templates
    - Email templates
    - SMS templates

### Integration Features
19. **Interoperability**
    - Payment gateway integration
    - Mobile money integration
    - Inter-bank transfers

20. **Batch Processing**
    - End-of-day processing
    - Interest calculation jobs
    - Scheduled reports
    - Bulk operations

### Advanced Features
21. **Surveys & Social Performance**
    - Poverty scoring (PPI)
    - Client surveys
    - Social performance metrics

22. **Float & Interest Rates**
    - Floating interest rates
    - Rate changes management

23. **Provisioning**
    - Provisioning criteria
    - Provisioning categories
    - Provision entries

24. **Audit & Compliance**
    - Audit trails
    - Maker-checker
    - Transaction limits
    - Compliance reports

## Performance Requirements

### Response Times
- Page load: < 2 seconds
- API calls: < 500ms
- Search results: < 1 second
- Report generation: < 5 seconds
- Bulk operations: Progress indication

### Scalability
- Support 10,000+ concurrent users
- Handle 1M+ clients
- Process 100K+ daily transactions
- Store 5+ years of data

### Optimization Strategies
1. **Frontend**
   - Code splitting by route
   - Lazy loading for components
   - Image optimization
   - Service worker caching
   - Virtual scrolling for lists
   - Debounced search
   - Optimistic UI updates

2. **API**
   - Pagination for all lists
   - Field filtering
   - Response compression
   - HTTP/2 multiplexing
   - Connection pooling

3. **Data Management**
   - IndexedDB for offline storage
   - React Query for caching
   - Background sync
   - Incremental data loading

## Next Steps

1. Set up the project structure
2. Configure authentication
3. Create base components
4. Implement core features (prioritized by business value)
5. Add comprehensive testing
6. Set up CI/CD pipeline
7. Deploy to staging environment

## Resources

- [Fineract API Documentation](https://demo.mifos.io/api-docs/apiLive.htm)
- [Next.js Documentation](https://nextjs.org/docs)
- [Ant Design Documentation](https://ant.design/)
- [React Query Documentation](https://tanstack.com/query/latest)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/)