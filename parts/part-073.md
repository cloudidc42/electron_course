# Part 73: Electron + GraphQL

## ใช้ GraphQL ใน Electron App

ในบทนี้เราจะ integrate Apollo Client ใน renderer, GraphQL subscriptions, local GraphQL server ใน main process, schema-first design และ optimistic updates

---

## 1. Local GraphQL Server ใน Main Process

```typescript
// src/main/graphqlServer.ts
import { createServer } from 'http'
import { ApolloServer } from '@apollo/server'
import { expressMiddleware } from '@apollo/server/express4'
import { makeExecutableSchema } from '@graphql-tools/schema'
import { WebSocketServer } from 'ws'
import { useServer } from 'graphql-ws/lib/use/ws'
import express from 'express'
import { PubSub } from 'graphql-subscriptions'
import { readdir, stat, readFile, writeFile } from 'fs/promises'
import { join, basename, extname } from 'path'
import os from 'os'

export const pubsub = new PubSub()

// GraphQL Schema
const typeDefs = `
  scalar DateTime
  scalar JSON

  type Query {
    files(path: String!): [FileEntry!]!
    file(path: String!): FileContent
    systemInfo: SystemInfo!
    recentFiles: [FileEntry!]!
    searchFiles(query: String!, root: String): [FileEntry!]!
  }

  type Mutation {
    createFile(path: String!, content: String!): FileEntry!
    updateFile(path: String!, content: String!): FileEntry!
    deleteFile(path: String!): Boolean!
    createDirectory(path: String!): FileEntry!
    renameFile(path: String!, newName: String!): FileEntry!
  }

  type Subscription {
    fileChanged(directory: String!): FileEvent!
    systemInfoUpdated: SystemInfo!
  }

  type FileEntry {
    name: String!
    path: String!
    type: FileType!
    size: Int!
    modified: DateTime!
    extension: String!
  }

  type FileContent {
    path: String!
    content: String!
    language: String!
  }

  enum FileType {
    file
    directory
    symlink
  }

  type FileEvent {
    type: FileEventType!
    path: String!
    file: FileEntry
  }

  enum FileEventType {
    add
    change
    unlink
    addDir
    unlinkDir
  }

  type SystemInfo {
    cpuUsage: Float!
    memoryUsed: Int!
    memoryTotal: Int!
    uptime: Int!
    platform: String!
  }
`

const resolvers = {
  Query: {
    files: async (_: unknown, { path }: { path: string }) => {
      const entries = await readdir(path, { withFileTypes: true })
      return Promise.all(entries.map(async (entry) => {
        const fullPath = join(path, entry.name)
        const stats = await stat(fullPath)
        return {
          name: entry.name,
          path: fullPath,
          type: entry.isDirectory() ? 'directory' : entry.isSymbolicLink() ? 'symlink' : 'file',
          size: stats.size,
          modified: stats.mtime,
          extension: extname(entry.name).slice(1).toLowerCase()
        }
      }))
    },

    file: async (_: unknown, { path }: { path: string }) => {
      const content = await readFile(path, 'utf-8')
      const ext = extname(path).slice(1).toLowerCase()
      const langMap: Record<string, string> = {
        ts: 'typescript', js: 'javascript', py: 'python',
        md: 'markdown', json: 'json', css: 'css', html: 'html'
      }
      return { path, content, language: langMap[ext] || 'plaintext' }
    },

    systemInfo: () => {
      const total = os.totalmem()
      const free = os.freemem()
      return {
        cpuUsage: Math.random() * 100, // simplified
        memoryUsed: total - free,
        memoryTotal: total,
        uptime: os.uptime(),
        platform: process.platform
      }
    },

    recentFiles: () => [],

    searchFiles: async (_: unknown, { query, root }: { query: string; root?: string }) => {
      const searchRoot = root || os.homedir()
      // simplified search
      return []
    }
  },

  Mutation: {
    createFile: async (_: unknown, { path: filePath, content }: { path: string; content: string }) => {
      await writeFile(filePath, content, 'utf-8')
      const stats = await stat(filePath)
      return {
        name: basename(filePath),
        path: filePath,
        type: 'file',
        size: stats.size,
        modified: stats.mtime,
        extension: extname(filePath).slice(1)
      }
    },

    updateFile: async (_: unknown, { path: filePath, content }: { path: string; content: string }) => {
      await writeFile(filePath, content, 'utf-8')
      const stats = await stat(filePath)
      pubsub.publish('FILE_CHANGED', {
        fileChanged: { type: 'change', path: filePath }
      })
      return {
        name: basename(filePath),
        path: filePath,
        type: 'file',
        size: stats.size,
        modified: stats.mtime,
        extension: extname(filePath).slice(1)
      }
    },

    deleteFile: async (_: unknown, { path: filePath }: { path: string }) => {
      const { unlink } = await import('fs/promises')
      await unlink(filePath)
      return true
    },

    createDirectory: async (_: unknown, { path: dirPath }: { path: string }) => {
      const { mkdir } = await import('fs/promises')
      await mkdir(dirPath, { recursive: true })
      const stats = await stat(dirPath)
      return {
        name: basename(dirPath),
        path: dirPath,
        type: 'directory',
        size: 0,
        modified: stats.mtime,
        extension: ''
      }
    },

    renameFile: async (_: unknown, { path: oldPath, newName }: { path: string; newName: string }) => {
      const { rename } = await import('fs/promises')
      const newPath = join(oldPath.split('/').slice(0, -1).join('/'), newName)
      await rename(oldPath, newPath)
      const stats = await stat(newPath)
      return {
        name: newName,
        path: newPath,
        type: 'file',
        size: stats.size,
        modified: stats.mtime,
        extension: extname(newName).slice(1)
      }
    }
  },

  Subscription: {
    fileChanged: {
      subscribe: (_: unknown, { directory }: { directory: string }) => {
        return pubsub.asyncIterator(['FILE_CHANGED'])
      }
    },

    systemInfoUpdated: {
      subscribe: () => {
        const interval = setInterval(() => {
          const total = os.totalmem()
          const free = os.freemem()
          pubsub.publish('SYSTEM_INFO', {
            systemInfoUpdated: {
              cpuUsage: Math.random() * 100,
              memoryUsed: total - free,
              memoryTotal: total,
              uptime: os.uptime(),
              platform: process.platform
            }
          })
        }, 2000)

        return {
          [Symbol.asyncIterator]() {
            return pubsub.asyncIterator(['SYSTEM_INFO'])
          }
        }
      }
    }
  }
}

export async function startGraphQLServer(port = 4000): Promise<void> {
  const schema = makeExecutableSchema({ typeDefs, resolvers })
  const app = express()
  app.use(express.json())

  const server = createServer(app)

  // WebSocket สำหรับ subscriptions
  const wsServer = new WebSocketServer({ server, path: '/graphql' })
  const serverCleanup = useServer({ schema }, wsServer)

  const apolloServer = new ApolloServer({
    schema,
    plugins: [{
      async serverWillStart() {
        return {
          async drainServer() {
            await serverCleanup.dispose()
          }
        }
      }
    }]
  })

  await apolloServer.start()
  app.use('/graphql', expressMiddleware(apolloServer))

  await new Promise<void>(resolve => server.listen(port, resolve))
  console.log(`GraphQL server ready at http://localhost:${port}/graphql`)
}
```

---

## 2. Apollo Client Setup (Renderer)

```typescript
// src/renderer/apollo/client.ts
import { ApolloClient, InMemoryCache, split, HttpLink, from } from '@apollo/client'
import { GraphQLWsLink } from '@apollo/client/link/subscriptions'
import { createClient } from 'graphql-ws'
import { getMainDefinition } from '@apollo/client/utilities'
import { onError } from '@apollo/client/link/error'

// Error handling link
const errorLink = onError(({ graphQLErrors, networkError, operation }) => {
  if (graphQLErrors) {
    graphQLErrors.forEach(({ message, path }) => {
      console.error(`GraphQL error in ${path?.join('.')}: ${message}`)
    })
  }
  if (networkError) {
    console.error(`Network error: ${networkError.message}`)
  }
})

// HTTP Link สำหรับ queries/mutations
const httpLink = new HttpLink({
  uri: 'http://localhost:4000/graphql'
})

// WebSocket Link สำหรับ subscriptions
const wsLink = new GraphQLWsLink(
  createClient({ url: 'ws://localhost:4000/graphql' })
)

// Split based on operation type
const splitLink = split(
  ({ query }) => {
    const definition = getMainDefinition(query)
    return (
      definition.kind === 'OperationDefinition' &&
      definition.operation === 'subscription'
    )
  },
  wsLink,
  httpLink
)

export const apolloClient = new ApolloClient({
  link: from([errorLink, splitLink]),
  cache: new InMemoryCache({
    typePolicies: {
      FileEntry: {
        keyFields: ['path']
      },
      Query: {
        fields: {
          files: {
            keyArgs: ['path'],
            merge(existing = [], incoming: unknown[]) {
              return incoming
            }
          }
        }
      }
    }
  }),
  defaultOptions: {
    watchQuery: { fetchPolicy: 'cache-and-network' },
    query: { fetchPolicy: 'network-only' }
  }
})
```

---

## 3. GraphQL Queries & Mutations

```typescript
// src/renderer/graphql/operations.ts
import { gql } from '@apollo/client'

export const GET_FILES = gql`
  query GetFiles($path: String!) {
    files(path: $path) {
      name
      path
      type
      size
      modified
      extension
    }
  }
`

export const GET_FILE_CONTENT = gql`
  query GetFileContent($path: String!) {
    file(path: $path) {
      path
      content
      language
    }
  }
`

export const CREATE_FILE = gql`
  mutation CreateFile($path: String!, $content: String!) {
    createFile(path: $path, content: $content) {
      name
      path
      type
      size
      modified
    }
  }
`

export const UPDATE_FILE = gql`
  mutation UpdateFile($path: String!, $content: String!) {
    updateFile(path: $path, content: $content) {
      name
      path
      size
      modified
    }
  }
`

export const DELETE_FILE = gql`
  mutation DeleteFile($path: String!) {
    deleteFile(path: $path)
  }
`

export const SYSTEM_INFO_SUBSCRIPTION = gql`
  subscription SystemInfoUpdated {
    systemInfoUpdated {
      cpuUsage
      memoryUsed
      memoryTotal
      uptime
      platform
    }
  }
`

export const FILE_CHANGED_SUBSCRIPTION = gql`
  subscription FileChanged($directory: String!) {
    fileChanged(directory: $directory) {
      type
      path
      file {
        name
        path
        type
        size
      }
    }
  }
`
```

---

## 4. React Components ที่ใช้ GraphQL

```tsx
// src/renderer/components/FileExplorer.tsx
import { useQuery, useMutation, useSubscription } from '@apollo/client'
import {
  GET_FILES, CREATE_FILE, DELETE_FILE,
  FILE_CHANGED_SUBSCRIPTION
} from '../graphql/operations'

interface FileExplorerProps {
  currentPath: string
  onNavigate: (path: string) => void
}

export function FileExplorer({ currentPath, onNavigate }: FileExplorerProps) {
  const { data, loading, error, refetch } = useQuery(GET_FILES, {
    variables: { path: currentPath },
    pollInterval: 0  // ใช้ subscription แทน
  })

  const [createFile] = useMutation(CREATE_FILE, {
    // Optimistic update
    optimisticResponse: ({ path, content }) => ({
      createFile: {
        __typename: 'FileEntry',
        name: path.split('/').pop(),
        path,
        type: 'file',
        size: content.length,
        modified: new Date().toISOString(),
        extension: path.split('.').pop() || ''
      }
    }),
    // Update cache
    update(cache, { data }) {
      const existing = cache.readQuery<{ files: unknown[] }>({
        query: GET_FILES,
        variables: { path: currentPath }
      })
      if (existing && data?.createFile) {
        cache.writeQuery({
          query: GET_FILES,
          variables: { path: currentPath },
          data: { files: [...existing.files, data.createFile] }
        })
      }
    }
  })

  const [deleteFile] = useMutation(DELETE_FILE, {
    update(cache, _, { variables }) {
      const existing = cache.readQuery<{ files: Array<{ path: string }> }>({
        query: GET_FILES,
        variables: { path: currentPath }
      })
      if (existing && variables?.path) {
        cache.writeQuery({
          query: GET_FILES,
          variables: { path: currentPath },
          data: { files: existing.files.filter(f => f.path !== variables.path) }
        })
      }
    }
  })

  // Subscription สำหรับ real-time updates
  useSubscription(FILE_CHANGED_SUBSCRIPTION, {
    variables: { directory: currentPath },
    onData: ({ data: { data: eventData } }) => {
      if (eventData?.fileChanged) {
        refetch()
      }
    }
  })

  if (loading) return <div>กำลังโหลด...</div>
  if (error) return <div>ข้อผิดพลาด: {error.message}</div>

  return (
    <div className="file-explorer">
      <div className="toolbar">
        <button onClick={() => {
          const name = prompt('ชื่อไฟล์ใหม่:')
          if (name) {
            createFile({
              variables: { path: `${currentPath}/${name}`, content: '' }
            })
          }
        }}>
          + ไฟล์ใหม่
        </button>
      </div>

      <div className="file-list">
        {data?.files.map((file: { name: string; path: string; type: string; size: number }) => (
          <div key={file.path} className="file-item">
            <span
              className="file-name"
              onClick={() => file.type === 'directory' ? onNavigate(file.path) : null}
            >
              {file.type === 'directory' ? '📁' : '📄'} {file.name}
            </span>
            <span className="file-size">{file.size}</span>
            <button
              onClick={() => {
                if (confirm(`ลบ ${file.name}?`)) {
                  deleteFile({ variables: { path: file.path } })
                }
              }}
            >
              🗑
            </button>
          </div>
        ))}
      </div>
    </div>
  )
}
```

---

## 5. System Monitor ด้วย Subscription

```tsx
// src/renderer/components/SystemMonitorGQL.tsx
import { useSubscription, useQuery } from '@apollo/client'
import { SYSTEM_INFO_SUBSCRIPTION } from '../graphql/operations'
import { useState, useEffect } from 'react'
import { gql } from '@apollo/client'

const GET_SYSTEM_INFO = gql`
  query GetSystemInfo {
    systemInfo {
      cpuUsage
      memoryUsed
      memoryTotal
      uptime
      platform
    }
  }
`

export function SystemMonitor() {
  const [history, setHistory] = useState<number[]>([])

  const { data: initialData } = useQuery(GET_SYSTEM_INFO)

  const { data: liveData } = useSubscription(SYSTEM_INFO_SUBSCRIPTION, {
    onData: ({ data: { data } }) => {
      if (data?.systemInfoUpdated) {
        setHistory(prev => [...prev, data.systemInfoUpdated.cpuUsage].slice(-60))
      }
    }
  })

  const info = liveData?.systemInfoUpdated || initialData?.systemInfo

  if (!info) return <div>กำลังโหลด...</div>

  const memPercent = (info.memoryUsed / info.memoryTotal * 100).toFixed(1)

  return (
    <div className="system-monitor">
      <div className="stat">
        <label>CPU</label>
        <div className="bar">
          <div style={{ width: `${info.cpuUsage}%`, background: '#e74c3c' }} />
        </div>
        <span>{info.cpuUsage.toFixed(1)}%</span>
      </div>
      <div className="stat">
        <label>RAM</label>
        <div className="bar">
          <div style={{ width: `${memPercent}%`, background: '#3498db' }} />
        </div>
        <span>{memPercent}%</span>
      </div>
      <div className="history">
        {history.map((v, i) => (
          <div key={i} className="bar-cell" style={{ height: `${v}%`, background: '#e74c3c' }} />
        ))}
      </div>
    </div>
  )
}
```

---

## สรุป

| Feature | รายละเอียด |
|---------|------------|
| Local GraphQL server | Express + Apollo Server 4 |
| WebSocket subscriptions | graphql-ws |
| Apollo Client | Caching + normalized store |
| Optimistic updates | UI updates ก่อน server confirm |
| Cache management | Automatic + manual updates |
| Schema-first | Type-safe resolvers |
| Real-time | File changes + system info |
