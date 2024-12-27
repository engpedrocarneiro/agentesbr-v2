# Migração Firebase para Supabase + Implementação RAG

## Índice
1. [Preparação do Ambiente](#1-preparação-do-ambiente)
2. [Configuração Supabase](#2-configuração-supabase)
3. [Estrutura do Banco de Dados](#3-estrutura-do-banco-de-dados)
4. [Migração de Dados](#4-migração-de-dados)
5. [Atualização do Frontend](#5-atualização-do-frontend)
6. [Sistema de Documentos](#6-sistema-de-documentos)
7. [Implementação RAG](#7-implementação-rag)
8. [Testes e Validação](#8-testes-e-validação)
9. [Deploy](#9-deploy)

## 1. Preparação do Ambiente

### 1.1 Novo Repositório
```bash
# Criar novo repositório
git clone https://github.com/[usuario]/agentesbr-v2.git
cd agentesbr-v2

# Copiar código atual
cp -r ../agentesbr/* .
rm -rf .git
git init
git add .
git commit -m "Initial commit: Base code from agentesbr"
```

### 1.2 Dependências
```json
// package.json - Novas dependências
{
  "dependencies": {
    "@supabase/supabase-js": "^2.x.x",
    "langchain": "^0.x.x",
    "pdf-parse": "^1.x.x",
    "openai": "^4.x.x"
  }
}
```

## 2. Configuração Supabase

### 2.1 Projeto Supabase
1. Criar novo projeto no Supabase
2. Guardar credenciais:
   - Project URL
   - anon key
   - service_role key

### 2.2 Configuração de Autenticação
```sql
-- Habilitar auth providers
create policy "Enable email/password auth"
on auth.users
for select
using (true);

-- Configurar email templates
update auth.templates
set template = '...'
where type = 'confirmation';
```

### 2.3 Storage Buckets
```sql
-- Criar buckets
create policy "Documentos acessíveis por usuário"
on storage.objects
for select
using (auth.uid() = owner_id);

-- Bucket para documentos
insert into storage.buckets (id, name)
values ('documents', 'Documentos dos Usuários');
```

## 3. Estrutura do Banco de Dados

### 3.1 Tabelas Principais
```sql
-- Usuários estendidos
create table public.profiles (
  id uuid references auth.users primary key,
  email text unique not null,
  full_name text,
  company_name text,
  created_at timestamp with time zone default timezone('utc'::text, now()) not null
);

-- Documentos
create table public.documents (
  id uuid default uuid_generate_v4() primary key,
  user_id uuid references auth.users not null,
  title text not null,
  description text,
  file_path text not null,
  mime_type text not null,
  status text default 'pending' not null,
  created_at timestamp with time zone default timezone('utc'::text, now()) not null
);

-- Chunks processados
create table public.document_chunks (
  id uuid default uuid_generate_v4() primary key,
  document_id uuid references public.documents not null,
  content text not null,
  embedding vector(1536),
  metadata jsonb,
  created_at timestamp with time zone default timezone('utc'::text, now()) not null
);
```

### 3.2 Políticas de Segurança
```sql
-- RLS para documentos
alter table public.documents enable row level security;

create policy "Documentos visíveis apenas para proprietário"
on public.documents for all
using (auth.uid() = user_id);

-- RLS para chunks
alter table public.document_chunks enable row level security;

create policy "Chunks visíveis apenas para proprietário do documento"
on public.document_chunks for all
using (
  exists (
    select 1 from public.documents
    where documents.id = document_chunks.document_id
    and documents.user_id = auth.uid()
  )
);
```

## 4. Migração de Dados

### 4.1 Script de Migração de Usuários
```typescript
// scripts/migrate-users.ts
import { supabase } from '../src/lib/supabase';
import { auth } from '../src/services/firebase';

async function migrateUsers() {
  const firebaseUsers = await auth.listUsers();
  
  for (const user of firebaseUsers.users) {
    // Criar usuário no Supabase
    const { data, error } = await supabase.auth.admin.createUser({
      email: user.email,
      password: generateTempPassword(),
      email_confirm: true
    });
    
    if (error) console.error(`Erro ao migrar ${user.email}:`, error);
    else console.log(`Migrado: ${user.email}`);
    
    // Criar perfil estendido
    await supabase.from('profiles').insert({
      id: data.user.id,
      email: user.email,
      // ... outros dados
    });
  }
}
```

### 4.2 Migração de Arquivos
```typescript
// scripts/migrate-files.ts
import { storage as firebaseStorage } from '../src/services/firebase';
import { supabase } from '../src/lib/supabase';

async function migrateFiles() {
  const files = await firebaseStorage.list();
  
  for (const file of files) {
    const downloadUrl = await file.getDownloadURL();
    const response = await fetch(downloadUrl);
    const blob = await response.blob();
    
    await supabase.storage
      .from('documents')
      .upload(`migrated/${file.name}`, blob);
  }
}
```

## 5. Atualização do Frontend

### 5.1 Cliente Supabase
```typescript
// src/lib/supabase.ts
import { createClient } from '@supabase/supabase-js';

export const supabase = createClient(
  process.env.VITE_SUPABASE_URL!,
  process.env.VITE_SUPABASE_ANON_KEY!
);
```

### 5.2 Hook de Autenticação
```typescript
// src/hooks/useAuth.ts
import { useEffect } from 'react';
import { useNavigate } from 'react-router-dom';
import { supabase } from '../lib/supabase';

export function useAuth() {
  const navigate = useNavigate();
  
  useEffect(() => {
    const { data: { subscription } } = supabase.auth.onAuthStateChange(
      (event, session) => {
        if (event === 'SIGNED_OUT') navigate('/');
      }
    );
    
    return () => subscription.unsubscribe();
  }, [navigate]);
  
  return {
    signIn: (email: string, password: string) =>
      supabase.auth.signInWithPassword({ email, password }),
    signUp: (email: string, password: string) =>
      supabase.auth.signUp({ email, password }),
    signOut: () => supabase.auth.signOut()
  };
}
```

## 6. Sistema de Documentos

### 6.1 Componente de Upload
```typescript
// src/components/features/DocumentUpload.tsx
import { useState } from 'react';
import { supabase } from '../../lib/supabase';

export function DocumentUpload() {
  const [uploading, setUploading] = useState(false);
  
  async function handleUpload(file: File) {
    setUploading(true);
    
    try {
      // Upload arquivo
      const { data: fileData, error: uploadError } = await supabase.storage
        .from('documents')
        .upload(`${user.id}/${file.name}`, file);
      
      if (uploadError) throw uploadError;
      
      // Criar registro
      const { error: dbError } = await supabase
        .from('documents')
        .insert({
          title: file.name,
          file_path: fileData.path,
          mime_type: file.type,
          user_id: user.id
        });
      
      if (dbError) throw dbError;
      
      // Iniciar processamento
      await processDocument(fileData.path);
    } catch (error) {
      console.error('Erro no upload:', error);
    } finally {
      setUploading(false);
    }
  }
  
  return (
    <div>
      <input
        type="file"
        accept=".pdf,.doc,.docx,.txt"
        onChange={e => handleUpload(e.target.files[0])}
        disabled={uploading}
      />
    </div>
  );
}
```

### 6.2 Processamento de Documentos
```typescript
// src/services/documents.ts
import { supabase } from '../lib/supabase';
import { OpenAI } from 'langchain/llms/openai';
import { PDFLoader } from 'langchain/document_loaders/fs/pdf';

export async function processDocument(filePath: string) {
  // Download do arquivo
  const { data: fileData } = await supabase.storage
    .from('documents')
    .download(filePath);
    
  // Extrair texto
  const loader = new PDFLoader(fileData);
  const docs = await loader.load();
  
  // Processar com LLM
  const llm = new OpenAI({
    temperature: 0,
    modelName: 'gpt-4'
  });
  
  // Criar chunks e embeddings
  for (const doc of docs) {
    const chunks = splitIntoChunks(doc.pageContent);
    
    for (const chunk of chunks) {
      const embedding = await generateEmbedding(chunk);
      
      await supabase.from('document_chunks').insert({
        document_id: doc.metadata.documentId,
        content: chunk,
        embedding,
        metadata: doc.metadata
      });
    }
  }
}
```

## 7. Implementação RAG

### 7.1 Serviço RAG
```typescript
// src/services/rag.ts
import { supabase } from '../lib/supabase';
import { OpenAIEmbeddings } from 'langchain/embeddings/openai';

export async function queryDocuments(query: string, userId: string) {
  // Gerar embedding da query
  const embeddings = new OpenAIEmbeddings();
  const queryEmbedding = await embeddings.embedQuery(query);
  
  // Buscar chunks relevantes
  const { data: chunks } = await supabase.rpc('match_documents', {
    query_embedding: queryEmbedding,
    match_threshold: 0.8,
    match_count: 5,
    user_id: userId
  });
  
  return chunks;
}
```

### 7.2 Integração com Chat
```typescript
// src/services/chat/processor.ts
import { queryDocuments } from '../rag';

class ChatProcessor {
  async process(messages, context, agentName, temperature) {
    // Buscar documentos relevantes
    const relevantDocs = await queryDocuments(
      messages[messages.length - 1].text,
      context.userId
    );
    
    // Adicionar ao contexto
    const enhancedContext = {
      ...context,
      documents: relevantDocs
    };
    
    // Processar resposta
    const response = await generateResponse(
      messages,
      temperature,
      enhancedContext,
      agentName
    );
    
    return response;
  }
}
```

## 8. Testes e Validação

### 8.1 Testes de Migração
- Verificar integridade dos dados migrados
- Validar permissões e políticas
- Testar autenticação e sessões

### 8.2 Testes de Funcionalidade
- Upload e processamento de documentos
- Busca e relevância do RAG
- Performance do sistema

### 8.3 Testes de Segurança
- Isolamento de dados entre usuários
- Proteção de rotas
- Validação de tokens

## 9. Deploy

### 9.1 Configuração de Ambiente
```bash
# .env.production
VITE_SUPABASE_URL=https://seu-projeto.supabase.co
VITE_SUPABASE_ANON_KEY=sua-chave-anon
VITE_OPENAI_API_KEY=sua-chave-openai
```

### 9.2 Build e Deploy
```bash
# Build do projeto
npm run build

# Deploy (exemplo com Vercel)
vercel deploy
```

### 9.3 Monitoramento
- Configurar logs e alertas
- Monitorar uso de recursos
- Acompanhar métricas de performance

## Notas Importantes

1. Backup
- Realizar backup completo antes da migração
- Manter Firebase ativo durante transição
- Validar dados antes de desativar Firebase

2. Segurança
- Nunca commitar chaves de API
- Usar variáveis de ambiente
- Implementar rate limiting

3. Performance
- Otimizar tamanho dos chunks
- Implementar cache de embeddings
- Monitorar custos de API

4. Manutenção
- Documentar todas as alterações
- Manter logs de migração
- Planejar rollback se necessário
