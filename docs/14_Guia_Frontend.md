# 14. Guia de Estrutura Frontend - App View v2.0

**Versão:** 1.0  
**Última Atualização:** Janeiro 2025  
**Responsável:** Arquitetura Frontend

---

## Sumário

1. [Stack Tecnológico](#stack-tecnologico)
2. [Arquitetura Feature-First](#arquitetura-feature-first)
3. [Estrutura de Pastas](#estrutura-de-pastas)
4. [Design System](#design-system)
5. [Gerenciamento de Estado](#gerenciamento-de-estado)
6. [Data Fetching com Tanstack Query](#data-fetching-com-tanstack-query)
7. [Otimizações de Performance](#otimizacoes-de-performance)
8. [Testes](#testes)
9. [Padrões e Convenções](#padroes-e-convencoes)
10. [Exemplos Práticos](#exemplos-praticos)

---

## Stack Tecnológico

### Core

| Tecnologia | Versão | Propósito |
|------------|--------|-----------|
| **React** | 18.3+ | Biblioteca UI |
| **Next.js** | 14+ (App Router) | Framework SSR/SSG |
| **TypeScript** | 5.3+ | Type safety |
| **TailwindCSS** | 3.4+ | Styling |
| **ShadCN UI** | latest | Design System base |
| **Tanstack Query** | 5.x | Data fetching/cache |
| **Playwright** | latest | Testes E2E |
| **Vitest** | latest | Unit/Integration tests |

### Bibliotecas Auxiliares

```json
{
  "dependencies": {
    "react": "^18.3.0",
    "next": "^14.2.0",
    "typescript": "^5.3.0",
    "tailwindcss": "^3.4.0",
    "@tanstack/react-query": "^5.0.0",
    "zod": "^3.22.0",
    "react-hook-form": "^7.49.0",
    "date-fns": "^3.0.0",
    "lucide-react": "^0.300.0"
  },
  "devDependencies": {
    "@playwright/test": "^1.40.0",
    "vitest": "^1.0.0",
    "@testing-library/react": "^14.1.0",
    "@types/node": "^20.10.0",
    "@types/react": "^18.2.0",
    "eslint": "^8.55.0",
    "prettier": "^3.1.0"
  }
}
```

---

## Arquitetura Feature-First

### Princípios

A arquitetura **feature-first** organiza o código por **funcionalidades de negócio**, não por tipos técnicos.

**❌ Evitar (organização por tipo técnico):**
```
src/
  components/
    SinistroCard.tsx
    ClienteList.tsx
    PericiaForm.tsx
  hooks/
    useSinistro.ts
    useCliente.ts
  services/
    sinistroService.ts
    clienteService.ts
```

**✅ Preferir (organização por feature):**
```
src/
  features/
    sinistro/
      components/
      hooks/
      services/
      types/
    cliente/
      components/
      hooks/
      services/
      types/
```

### Benefícios

- ✅ **Coesão alta:** Tudo relacionado a uma feature está junto
- ✅ **Acoplamento baixo:** Features isoladas, fácil mover/deletar
- ✅ **Escalabilidade:** Adicionar novas features não afeta as existentes
- ✅ **Colaboração:** Times podem trabalhar em features diferentes sem conflitos

---

## Estrutura de Pastas

```
appview-frontend/
├── src/
│   ├── app/                          # Next.js App Router
│   │   ├── (auth)/                   # Route group - autenticação
│   │   │   ├── login/
│   │   │   │   └── page.tsx
│   │   │   └── layout.tsx
│   │   ├── (dashboard)/              # Route group - dashboard
│   │   │   ├── sinistros/
│   │   │   │   ├── page.tsx          # /sinistros
│   │   │   │   ├── [id]/
│   │   │   │   │   └── page.tsx      # /sinistros/[id]
│   │   │   │   └── novo/
│   │   │   │       └── page.tsx      # /sinistros/novo
│   │   │   ├── clientes/
│   │   │   ├── pericias/
│   │   │   └── layout.tsx
│   │   ├── api/                      # API Routes (se necessário)
│   │   │   └── auth/
│   │   │       └── [...nextauth]/
│   │   │           └── route.ts
│   │   ├── layout.tsx                # Root layout
│   │   └── page.tsx                  # Home page
│   │
│   ├── features/                     # Features (feature-first)
│   │   ├── sinistro/
│   │   │   ├── components/
│   │   │   │   ├── sinistro-card.tsx
│   │   │   │   ├── sinistro-form.tsx
│   │   │   │   ├── sinistro-list.tsx
│   │   │   │   └── sinistro-details.tsx
│   │   │   ├── hooks/
│   │   │   │   ├── use-sinistros.ts
│   │   │   │   ├── use-create-sinistro.ts
│   │   │   │   └── use-update-sinistro.ts
│   │   │   ├── services/
│   │   │   │   └── sinistro-api.ts
│   │   │   ├── types/
│   │   │   │   └── sinistro.types.ts
│   │   │   └── index.ts              # Exports públicos
│   │   │
│   │   ├── cliente/
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── services/
│   │   │   ├── types/
│   │   │   └── index.ts
│   │   │
│   │   ├── pericia/
│   │   ├── documento/
│   │   └── auth/
│   │
│   ├── design-system/                # Design System
│   │   ├── components/
│   │   │   ├── ui/                   # ShadCN components
│   │   │   │   ├── button.tsx
│   │   │   │   ├── input.tsx
│   │   │   │   ├── dialog.tsx
│   │   │   │   ├── card.tsx
│   │   │   │   ├── table.tsx
│   │   │   │   └── ...
│   │   │   ├── form/                 # Form components
│   │   │   │   ├── text-field.tsx
│   │   │   │   ├── select-field.tsx
│   │   │   │   └── date-picker.tsx
│   │   │   ├── layout/               # Layout components
│   │   │   │   ├── sidebar.tsx
│   │   │   │   ├── header.tsx
│   │   │   │   └── container.tsx
│   │   │   └── feedback/             # Feedback components
│   │   │       ├── toast.tsx
│   │   │       ├── loading.tsx
│   │   │       └── error-boundary.tsx
│   │   └── index.ts
│   │
│   ├── lib/                          # Utilities & configs
│   │   ├── api-client.ts             # HTTP client (fetch wrapper)
│   │   ├── query-client.ts           # Tanstack Query config
│   │   ├── auth.ts                   # Auth helpers
│   │   ├── utils.ts                  # Utility functions
│   │   └── constants.ts              # Global constants
│   │
│   ├── hooks/                        # Shared hooks
│   │   ├── use-toast.ts
│   │   ├── use-debounce.ts
│   │   ├── use-local-storage.ts
│   │   └── use-media-query.ts
│   │
│   ├── types/                        # Shared types
│   │   ├── api.types.ts
│   │   ├── common.types.ts
│   │   └── env.d.ts
│   │
│   └── styles/
│       └── globals.css               # Tailwind imports
│
├── public/
│   ├── images/
│   └── icons/
│
├── e2e/                              # Playwright E2E tests
│   ├── sinistro.spec.ts
│   ├── auth.spec.ts
│   └── playwright.config.ts
│
├── .env.local                        # Environment variables
├── .eslintrc.json                    # ESLint config
├── .prettierrc                       # Prettier config
├── next.config.js                    # Next.js config
├── tailwind.config.ts                # Tailwind config
├── tsconfig.json                     # TypeScript config
├── vitest.config.ts                  # Vitest config
└── package.json
```

---

## Design System

### Componentes Reutilizáveis (ShadCN)

O **ShadCN UI** é a base do design system. Componentes são copiados para o projeto (não instalados via npm), permitindo customização total.

#### Instalação de Componentes

```bash
# Instalar ShadCN CLI
npx shadcn-ui@latest init

# Adicionar componentes conforme necessário
npx shadcn-ui@latest add button
npx shadcn-ui@latest add input
npx shadcn-ui@latest add dialog
npx shadcn-ui@latest add card
npx shadcn-ui@latest add table
npx shadcn-ui@latest add form
```

#### Estrutura do Design System

```tsx
// src/design-system/components/ui/button.tsx
import * as React from "react"
import { Slot } from "@radix-ui/react-slot"
import { cva, type VariantProps } from "class-variance-authority"
import { cn } from "@/lib/utils"

const buttonVariants = cva(
  "inline-flex items-center justify-center rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive: "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline: "border border-input bg-background hover:bg-accent hover:text-accent-foreground",
        secondary: "bg-secondary text-secondary-foreground hover:bg-secondary/80",
        ghost: "hover:bg-accent hover:text-accent-foreground",
        link: "text-primary underline-offset-4 hover:underline",
      },
      size: {
        default: "h-10 px-4 py-2",
        sm: "h-9 rounded-md px-3",
        lg: "h-11 rounded-md px-8",
        icon: "h-10 w-10",
      },
    },
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  }
)

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean
}

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    const Comp = asChild ? Slot : "button"
    return (
      <Comp
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        {...props}
      />
    )
  }
)
Button.displayName = "Button"

export { Button, buttonVariants }
```

#### Exemplo de Uso

```tsx
import { Button } from "@/design-system/components/ui/button"

export function MyComponent() {
  return (
    <>
      <Button>Default</Button>
      <Button variant="destructive">Delete</Button>
      <Button variant="outline" size="lg">Large</Button>
    </>
  )
}
```

### Componentes Customizados

Criar componentes específicos do domínio que USAM os componentes base:

```tsx
// src/design-system/components/form/text-field.tsx
import { Input } from "@/design-system/components/ui/input"
import { Label } from "@/design-system/components/ui/label"
import { cn } from "@/lib/utils"

interface TextFieldProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string
  error?: string
  helperText?: string
}

export function TextField({ 
  label, 
  error, 
  helperText, 
  className, 
  ...props 
}: TextFieldProps) {
  return (
    <div className="space-y-2">
      <Label htmlFor={props.id}>{label}</Label>
      <Input
        className={cn(error && "border-destructive", className)}
        {...props}
      />
      {error && (
        <p className="text-sm text-destructive">{error}</p>
      )}
      {helperText && !error && (
        <p className="text-sm text-muted-foreground">{helperText}</p>
      )}
    </div>
  )
}
```

---

## Gerenciamento de Estado

### Estratégia Híbrida

| Tipo de Estado | Solução | Quando Usar |
|----------------|---------|-------------|
| **Server State** | Tanstack Query | Dados da API (sinistros, clientes) |
| **Local State** | useState | Estado de componente (formulário, toggle) |
| **Shared State** | Context API | Tema, autenticação, sidebar |
| **Form State** | React Hook Form | Formulários complexos |

### Exemplo: Context para Auth

```tsx
// src/features/auth/context/auth-context.tsx
'use client'

import { createContext, useContext, useState, useCallback } from 'react'

interface User {
  id: string
  name: string
  email: string
  roles: string[]
}

interface AuthContextType {
  user: User | null
  isAuthenticated: boolean
  login: (email: string, password: string) => Promise<void>
  logout: () => void
}

const AuthContext = createContext<AuthContextType | undefined>(undefined)

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null)

  const login = useCallback(async (email: string, password: string) => {
    // Lógica de login
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      body: JSON.stringify({ email, password }),
    })
    const data = await response.json()
    setUser(data.user)
  }, [])

  const logout = useCallback(() => {
    setUser(null)
  }, [])

  return (
    <AuthContext.Provider
      value={{
        user,
        isAuthenticated: !!user,
        login,
        logout,
      }}
    >
      {children}
    </AuthContext.Provider>
  )
}

export function useAuth() {
  const context = useContext(AuthContext)
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider')
  }
  return context
}
```

---

## Data Fetching com Tanstack Query

### Configuração

```tsx
// src/lib/query-client.ts
import { QueryClient } from '@tanstack/react-query'

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5 minutos
      gcTime: 1000 * 60 * 10, // 10 minutos
      retry: 3,
      refetchOnWindowFocus: false,
    },
  },
})
```

```tsx
// src/app/layout.tsx
'use client'

import { QueryClientProvider } from '@tanstack/react-query'
import { queryClient } from '@/lib/query-client'

export default function RootLayout({ children }) {
  return (
    <html lang="pt-BR">
      <body>
        <QueryClientProvider client={queryClient}>
          {children}
        </QueryClientProvider>
      </body>
    </html>
  )
}
```

### Exemplo: Feature Sinistro

#### 1. API Service

```tsx
// src/features/sinistro/services/sinistro-api.ts
import { apiClient } from '@/lib/api-client'
import type { Sinistro, CreateSinistroDTO } from '../types/sinistro.types'

export const sinistroApi = {
  getAll: async (): Promise<Sinistro[]> => {
    const response = await apiClient.get('/sinistros')
    return response.data
  },

  getById: async (id: string): Promise<Sinistro> => {
    const response = await apiClient.get(`/sinistros/${id}`)
    return response.data
  },

  create: async (data: CreateSinistroDTO): Promise<Sinistro> => {
    const response = await apiClient.post('/sinistros', data)
    return response.data
  },

  update: async (id: string, data: Partial<Sinistro>): Promise<Sinistro> => {
    const response = await apiClient.put(`/sinistros/${id}`, data)
    return response.data
  },
}
```

#### 2. Custom Hooks

```tsx
// src/features/sinistro/hooks/use-sinistros.ts
import { useQuery } from '@tanstack/react-query'
import { sinistroApi } from '../services/sinistro-api'

export function useSinistros() {
  return useQuery({
    queryKey: ['sinistros'],
    queryFn: sinistroApi.getAll,
  })
}

export function useSinistro(id: string) {
  return useQuery({
    queryKey: ['sinistros', id],
    queryFn: () => sinistroApi.getById(id),
    enabled: !!id,
  })
}
```

```tsx
// src/features/sinistro/hooks/use-create-sinistro.ts
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { sinistroApi } from '../services/sinistro-api'
import { useToast } from '@/hooks/use-toast'

export function useCreateSinistro() {
  const queryClient = useQueryClient()
  const { toast } = useToast()

  return useMutation({
    mutationFn: sinistroApi.create,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['sinistros'] })
      toast({
        title: 'Sinistro criado',
        description: 'O sinistro foi criado com sucesso.',
      })
    },
    onError: (error) => {
      toast({
        title: 'Erro ao criar sinistro',
        description: error.message,
        variant: 'destructive',
      })
    },
  })
}
```

#### 3. Component

```tsx
// src/features/sinistro/components/sinistro-list.tsx
'use client'

import { useSinistros } from '../hooks/use-sinistros'
import { SinistroCard } from './sinistro-card'
import { Button } from '@/design-system/components/ui/button'
import Link from 'next/link'

export function SinistroList() {
  const { data: sinistros, isLoading, error } = useSinistros()

  if (isLoading) {
    return <div>Carregando...</div>
  }

  if (error) {
    return <div>Erro ao carregar sinistros: {error.message}</div>
  }

  return (
    <div className="space-y-4">
      <div className="flex justify-between items-center">
        <h1 className="text-2xl font-bold">Sinistros</h1>
        <Link href="/sinistros/novo">
          <Button>Novo Sinistro</Button>
        </Link>
      </div>

      <div className="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
        {sinistros?.map((sinistro) => (
          <SinistroCard key={sinistro.id} sinistro={sinistro} />
        ))}
      </div>
    </div>
  )
}
```

---

## Otimizações de Performance

### 1. useCallback

Usar `useCallback` para **callbacks passados para componentes filhos**:

```tsx
import { useCallback, useState } from 'react'

export function ParentComponent() {
  const [items, setItems] = useState([])

  // ✅ Bom: memoiza callback
  const handleDelete = useCallback((id: string) => {
    setItems(prev => prev.filter(item => item.id !== id))
  }, [])

  // ❌ Ruim: cria nova função a cada render
  // const handleDelete = (id: string) => {
  //   setItems(prev => prev.filter(item => item.id !== id))
  // }

  return (
    <>
      {items.map(item => (
        <ChildComponent 
          key={item.id} 
          item={item} 
          onDelete={handleDelete} 
        />
      ))}
    </>
  )
}
```

### 2. useMemo

Usar `useMemo` para **computações custosas**:

```tsx
import { useMemo } from 'react'

export function SinistroStats({ sinistros }) {
  // ✅ Bom: memoiza computação
  const stats = useMemo(() => {
    return {
      total: sinistros.length,
      abertos: sinistros.filter(s => s.status === 'ABERTO').length,
      fechados: sinistros.filter(s => s.status === 'FECHADO').length,
      valorTotal: sinistros.reduce((acc, s) => acc + s.valor, 0),
    }
  }, [sinistros])

  // ❌ Ruim: recomputa a cada render
  // const stats = {
  //   total: sinistros.length,
  //   abertos: sinistros.filter(s => s.status === 'ABERTO').length,
  //   ...
  // }

  return <div>{/* render stats */}</div>
}
```

### 3. React.memo

Usar `React.memo` para componentes que renderizam listas:

```tsx
import { memo } from 'react'

interface SinistroCardProps {
  sinistro: Sinistro
  onDelete: (id: string) => void
}

export const SinistroCard = memo(function SinistroCard({ 
  sinistro, 
  onDelete 
}: SinistroCardProps) {
  return (
    <div>
      <h3>{sinistro.numero}</h3>
      <button onClick={() => onDelete(sinistro.id)}>Deletar</button>
    </div>
  )
})
```

### 4. Next.js Image Optimization

```tsx
import Image from 'next/image'

export function Logo() {
  return (
    <Image
      src="/logo.png"
      alt="App View"
      width={200}
      height={50}
      priority // Carrega imediatamente
    />
  )
}
```

### 5. Dynamic Imports

```tsx
import dynamic from 'next/dynamic'

// Carrega componente apenas quando necessário
const HeavyChart = dynamic(() => import('./heavy-chart'), {
  loading: () => <p>Carregando gráfico...</p>,
  ssr: false, // Desabilita SSR para este componente
})

export function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      <HeavyChart />
    </div>
  )
}
```

---

## Testes

### Estratégia de Testes

| Tipo | Framework | Escopo | % Alvo |
|------|-----------|--------|--------|
| **E2E** | Playwright | Fluxos críticos | 10-15% |
| **Unit** | Vitest | Utils, hooks | 60-70% |
| **Integration** | Vitest + Testing Library | Componentes | 20-30% |

### Testes E2E (Playwright)

```typescript
// e2e/sinistro.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Sinistro Flow', () => {
  test.beforeEach(async ({ page }) => {
    // Login
    await page.goto('/login')
    await page.fill('[name="email"]', 'admin@appview.com')
    await page.fill('[name="password"]', 'admin123')
    await page.click('button[type="submit"]')
    await page.waitForURL('/dashboard')
  })

  test('deve criar novo sinistro', async ({ page }) => {
    // Navegar para página de criação
    await page.goto('/sinistros/novo')

    // Preencher formulário
    await page.fill('[name="cpf"]', '12345678900')
    await page.fill('[name="numeroContrato"]', 'CON-2025-001')
    await page.selectOption('[name="tipoSinistro"]', 'DANOS_MATERIAIS')
    await page.fill('[name="valorEstimado"]', '15000')
    await page.fill('[name="descricao"]', 'Colisão traseira')

    // Submit
    await page.click('button[type="submit"]')

    // Verificar sucesso
    await expect(page.locator('text=Sinistro criado com sucesso')).toBeVisible()
    await expect(page).toHaveURL(/\/sinistros\/\d+/)
  })

  test('deve validar CPF inválido', async ({ page }) => {
    await page.goto('/sinistros/novo')

    await page.fill('[name="cpf"]', '111.111.111-11')
    await page.click('button[type="submit"]')

    await expect(page.locator('text=CPF inválido')).toBeVisible()
  })

  test('deve anexar documento', async ({ page }) => {
    // Criar sinistro primeiro
    await page.goto('/sinistros/novo')
    // ... preencher formulário ...
    await page.click('button[type="submit"]')

    // Anexar documento
    const fileInput = page.locator('input[type="file"]')
    await fileInput.setInputFiles('./fixtures/boletim.pdf')

    await expect(page.locator('text=Documento enviado')).toBeVisible()
  })
})
```

### Testes de Componente (Vitest)

```tsx
// src/features/sinistro/components/__tests__/sinistro-card.test.tsx
import { describe, it, expect, vi } from 'vitest'
import { render, screen, fireEvent } from '@testing-library/react'
import { SinistroCard } from '../sinistro-card'

describe('SinistroCard', () => {
  const mockSinistro = {
    id: '1',
    numero: '2025000001',
    cpf: '12345678900',
    status: 'ABERTO',
    valorEstimado: 10000,
  }

  it('deve renderizar informações do sinistro', () => {
    render(<SinistroCard sinistro={mockSinistro} />)

    expect(screen.getByText('2025000001')).toBeInTheDocument()
    expect(screen.getByText('ABERTO')).toBeInTheDocument()
    expect(screen.getByText('R$ 10.000,00')).toBeInTheDocument()
  })

  it('deve chamar onDelete quando clicar no botão', () => {
    const onDelete = vi.fn()
    render(<SinistroCard sinistro={mockSinistro} onDelete={onDelete} />)

    fireEvent.click(screen.getByRole('button', { name: /deletar/i }))

    expect(onDelete).toHaveBeenCalledWith('1')
  })
})
```

### Testes de Hook (Vitest)

```tsx
// src/features/sinistro/hooks/__tests__/use-sinistros.test.ts
import { describe, it, expect } from 'vitest'
import { renderHook, waitFor } from '@testing-library/react'
import { QueryClientProvider } from '@tanstack/react-query'
import { useSinistros } from '../use-sinistros'
import { queryClient } from '@/lib/query-client'

const wrapper = ({ children }) => (
  <QueryClientProvider client={queryClient}>
    {children}
  </QueryClientProvider>
)

describe('useSinistros', () => {
  it('deve retornar lista de sinistros', async () => {
    const { result } = renderHook(() => useSinistros(), { wrapper })

    await waitFor(() => expect(result.current.isSuccess).toBe(true))

    expect(result.current.data).toHaveLength(3)
  })
})
```

### Configuração Vitest

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    setupFiles: ['./vitest.setup.ts'],
    globals: true,
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
})
```

```typescript
// vitest.setup.ts
import '@testing-library/jest-dom'
import { cleanup } from '@testing-library/react'
import { afterEach } from 'vitest'

afterEach(() => {
  cleanup()
})
```

---

## Padrões e Convenções

### 1. Nomenclatura de Arquivos

```
kebab-case para arquivos:
  sinistro-card.tsx
  use-sinistros.ts
  sinistro-api.ts

PascalCase para componentes:
  SinistroCard
  TextField
  Button
```

### 2. Estrutura de Componente

```tsx
// Imports externos
import { useState, useCallback } from 'react'
import { useQuery } from '@tanstack/react-query'

// Imports internos
import { Button } from '@/design-system/components/ui/button'
import { useSinistros } from '../hooks/use-sinistros'

// Types
interface SinistroListProps {
  filter?: string
}

// Component
export function SinistroList({ filter }: SinistroListProps) {
  // Hooks
  const { data, isLoading } = useSinistros()
  const [selected, setSelected] = useState<string | null>(null)

  // Callbacks
  const handleSelect = useCallback((id: string) => {
    setSelected(id)
  }, [])

  // Early returns
  if (isLoading) return <div>Loading...</div>

  // Render
  return (
    <div>
      {/* JSX */}
    </div>
  )
}
```

### 3. TypeScript

```tsx
// ✅ Usar tipos explícitos
interface User {
  id: string
  name: string
  email: string
}

// ✅ Evitar 'any'
function getUser(id: string): Promise<User> {
  // ...
}

// ✅ Usar enums para constantes
enum StatusSinistro {
  ABERTO = 'ABERTO',
  EM_ANALISE = 'EM_ANALISE',
  FECHADO = 'FECHADO',
}
```

### 4. Exports

```tsx
// ✅ Named exports (preferir)
export function SinistroCard() {}
export function SinistroList() {}

// ❌ Default exports (evitar)
// export default SinistroCard
```

---

## Exemplos Práticos

### Exemplo Completo: Feature Sinistro

#### 1. Types

```typescript
// src/features/sinistro/types/sinistro.types.ts
export interface Sinistro {
  id: string
  numero: string
  cpf: string
  numeroContrato: string
  tipoSinistro: TipoSinistro
  status: StatusSinistro
  valorEstimado: number
  descricao: string
  dataAbertura: string
  dataFechamento?: string
}

export enum TipoSinistro {
  DANOS_MATERIAIS = 'DANOS_MATERIAIS',
  DANOS_CORPORAIS = 'DANOS_CORPORAIS',
  ROUBO_FURTO = 'ROUBO_FURTO',
}

export enum StatusSinistro {
  ABERTO = 'ABERTO',
  EM_ANALISE = 'EM_ANALISE',
  AGUARDANDO_PERICIA = 'AGUARDANDO_PERICIA',
  FECHADO = 'FECHADO',
}

export interface CreateSinistroDTO {
  cpf: string
  numeroContrato: string
  tipoSinistro: TipoSinistro
  valorEstimado: number
  descricao: string
  dataOcorrencia: string
}
```

#### 2. Page (Next.js)

```tsx
// src/app/(dashboard)/sinistros/page.tsx
import { SinistroList } from '@/features/sinistro/components/sinistro-list'

export default function SinistrosPage() {
  return (
    <div className="container mx-auto py-8">
      <SinistroList />
    </div>
  )
}
```

#### 3. Component

```tsx
// src/features/sinistro/components/sinistro-form.tsx
'use client'

import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import * as z from 'zod'
import { Button } from '@/design-system/components/ui/button'
import { TextField } from '@/design-system/components/form/text-field'
import { useCreateSinistro } from '../hooks/use-create-sinistro'

const schema = z.object({
  cpf: z.string().regex(/^\d{11}$/, 'CPF inválido'),
  numeroContrato: z.string().min(1, 'Campo obrigatório'),
  tipoSinistro: z.enum(['DANOS_MATERIAIS', 'DANOS_CORPORAIS', 'ROUBO_FURTO']),
  valorEstimado: z.number().positive('Valor deve ser positivo'),
  descricao: z.string().min(10, 'Descrição muito curta'),
})

type FormData = z.infer<typeof schema>

export function SinistroForm() {
  const { mutate, isPending } = useCreateSinistro()
  const { register, handleSubmit, formState: { errors } } = useForm<FormData>({
    resolver: zodResolver(schema),
  })

  const onSubmit = (data: FormData) => {
    mutate(data)
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <TextField
        label="CPF"
        {...register('cpf')}
        error={errors.cpf?.message}
      />

      <TextField
        label="Número do Contrato"
        {...register('numeroContrato')}
        error={errors.numeroContrato?.message}
      />

      {/* Outros campos... */}

      <Button type="submit" disabled={isPending}>
        {isPending ? 'Criando...' : 'Criar Sinistro'}
      </Button>
    </form>
  )
}
```

---

## Checklist de Implementação

### Fase 1: Setup

- [ ] Criar projeto Next.js com TypeScript
- [ ] Configurar TailwindCSS
- [ ] Instalar ShadCN UI
- [ ] Configurar Tanstack Query
- [ ] Configurar Playwright
- [ ] Configurar Vitest

### Fase 2: Design System

- [ ] Adicionar componentes ShadCN base
- [ ] Criar componentes customizados (TextField, etc.)
- [ ] Criar layout components (Sidebar, Header)

### Fase 3: Features

- [ ] Implementar feature Sinistro
- [ ] Implementar feature Cliente
- [ ] Implementar feature Perícia
- [ ] Implementar feature Auth

### Fase 4: Testes

- [ ] Testes E2E dos fluxos críticos
- [ ] Testes de componentes principais
- [ ] Coverage > 70%

---

**Fim do Documento**
