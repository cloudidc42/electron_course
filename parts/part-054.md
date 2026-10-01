# Part 054: Electron with SQLite + ORM
## การใช้ SQLite กับ TypeORM, Prisma, และ Sequelize ใน Electron

---

## 🎯 เป้าหมายของบทเรียนนี้

- TypeORM กับ Electron
- Prisma กับ Electron (workaround)
- Sequelize
- Database migrations
- Relations
- Transactions
- Query optimization

---

## 1. TypeORM + better-sqlite3

### ติดตั้ง

```bash
npm install typeorm better-sqlite3 reflect-metadata
npm install --save-dev @types/better-sqlite3

# Rebuild สำหรับ Electron
npx @electron/rebuild -f -w better-sqlite3
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "strictPropertyInitialization": false
  }
}
```

---

## 2. Database Entities

### src/main/db/entities/Note.ts

```typescript
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  ManyToOne,
  OneToMany,
  Index,
  JoinColumn,
} from 'typeorm';

@Entity('notes')
export class Note {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'varchar', length: 255 })
  @Index()
  title: string;

  @Column({ type: 'text', nullable: true })
  content: string;

  @Column({ type: 'boolean', default: false })
  archived: boolean;

  @Column({ type: 'boolean', default: false })
  favorite: boolean;

  @Column({ type: 'json', nullable: true })
  tags: string[];

  @ManyToOne(() => Folder, folder => folder.notes, { nullable: true, onDelete: 'SET NULL' })
  @JoinColumn({ name: 'folder_id' })
  folder: Folder;

  @Column({ name: 'folder_id', nullable: true })
  folderId: string;

  @OneToMany(() => Attachment, attachment => attachment.note)
  attachments: Attachment[];

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;
}

@Entity('folders')
export class Folder {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'varchar', length: 100 })
  name: string;

  @Column({ type: 'varchar', length: 7, default: '#6366f1' })
  color: string;

  @ManyToOne(() => Folder, folder => folder.children, { nullable: true })
  parent: Folder;

  @OneToMany(() => Folder, folder => folder.parent)
  children: Folder[];

  @OneToMany(() => Note, note => note.folder)
  notes: Note[];

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;
}

@Entity('attachments')
export class Attachment {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  filename: string;

  @Column()
  path: string;

  @Column()
  mimeType: string;

  @Column()
  size: number;

  @ManyToOne(() => Note, note => note.attachments, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'note_id' })
  note: Note;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;
}
```

---

## 3. Database Setup

### src/main/db/database.ts

```typescript
import { DataSource, DataSourceOptions } from 'typeorm';
import { join } from 'path';
import { app } from 'electron';
import { Note, Folder, Attachment } from './entities/Note';

const DB_PATH = join(app.getPath('userData'), 'app.db');

const dbConfig: DataSourceOptions = {
  type: 'better-sqlite3',
  database: DB_PATH,
  
  entities: [Note, Folder, Attachment],
  
  // Migrations
  migrations: [join(__dirname, 'migrations/*.js')],
  migrationsRun: true,
  
  // Logging
  logging: process.env.NODE_ENV === 'development',
  logger: 'advanced-console',
  
  // SQLite-specific
  synchronize: false, // ใช้ migrations แทน
  
  // Performance
  extra: {
    // WAL mode สำหรับ performance ดีขึ้น
    pragma: {
      journal_mode: 'WAL',
      synchronous: 'NORMAL',
      cache_size: -64000, // 64MB
      page_size: 4096,
    },
  },
};

export const AppDataSource = new DataSource(dbConfig);

export async function initDatabase(): Promise<void> {
  try {
    await AppDataSource.initialize();
    console.log('Database initialized:', DB_PATH);
    
    // รัน migrations
    await AppDataSource.runMigrations();
    console.log('Migrations complete');
  } catch (error) {
    console.error('Database initialization failed:', error);
    throw error;
  }
}

export async function closeDatabase(): Promise<void> {
  if (AppDataSource.isInitialized) {
    await AppDataSource.destroy();
    console.log('Database closed');
  }
}
```

---

## 4. Repository Pattern

### src/main/db/repositories/NoteRepository.ts

```typescript
import { Repository, Like, FindOptionsWhere, Between, In } from 'typeorm';
import { AppDataSource } from '../database';
import { Note } from '../entities/Note';

export class NoteRepository {
  private repo: Repository<Note>;
  
  constructor() {
    this.repo = AppDataSource.getRepository(Note);
  }
  
  // ==========================================
  // CRUD
  // ==========================================
  
  async findAll(options?: {
    archived?: boolean;
    favorite?: boolean;
    folderId?: string;
    limit?: number;
    offset?: number;
  }): Promise<{ notes: Note[]; total: number }> {
    const where: FindOptionsWhere<Note> = {};
    
    if (options?.archived !== undefined) where.archived = options.archived;
    if (options?.favorite !== undefined) where.favorite = options.favorite;
    if (options?.folderId) where.folderId = options.folderId;
    
    const [notes, total] = await this.repo.findAndCount({
      where,
      relations: ['folder', 'attachments'],
      order: { updatedAt: 'DESC' },
      take: options?.limit || 50,
      skip: options?.offset || 0,
    });
    
    return { notes, total };
  }
  
  async findById(id: string): Promise<Note | null> {
    return this.repo.findOne({
      where: { id },
      relations: ['folder', 'attachments'],
    });
  }
  
  async create(data: Partial<Note>): Promise<Note> {
    const note = this.repo.create(data);
    return this.repo.save(note);
  }
  
  async update(id: string, data: Partial<Note>): Promise<Note | null> {
    await this.repo.update(id, data);
    return this.findById(id);
  }
  
  async delete(id: string): Promise<void> {
    await this.repo.delete(id);
  }
  
  async deleteMany(ids: string[]): Promise<void> {
    await this.repo.delete({ id: In(ids) });
  }
  
  // ==========================================
  // Search
  // ==========================================
  
  async search(query: string, limit: number = 20): Promise<Note[]> {
    return this.repo.createQueryBuilder('note')
      .where('note.title LIKE :query OR note.content LIKE :query', {
        query: `%${query}%`,
      })
      .andWhere('note.archived = :archived', { archived: false })
      .orderBy('note.updatedAt', 'DESC')
      .take(limit)
      .getMany();
  }
  
  // Full text search ด้วย FTS5 (SQLite)
  async fullTextSearch(query: string): Promise<Note[]> {
    return this.repo.query(`
      SELECT n.* FROM notes n
      INNER JOIN notes_fts fts ON fts.rowid = n.rowid
      WHERE notes_fts MATCH ?
      ORDER BY rank
      LIMIT 20
    `, [query]);
  }
  
  // ==========================================
  // Statistics
  // ==========================================
  
  async getStats(): Promise<{
    total: number;
    archived: number;
    favorites: number;
    byFolder: Record<string, number>;
  }> {
    const [total, archived, favorites] = await Promise.all([
      this.repo.count(),
      this.repo.count({ where: { archived: true } }),
      this.repo.count({ where: { favorite: true } }),
    ]);
    
    const byFolderRaw = await this.repo.createQueryBuilder('note')
      .select('note.folderId, COUNT(*) as count')
      .groupBy('note.folderId')
      .getRawMany();
    
    const byFolder = Object.fromEntries(
      byFolderRaw.map(r => [r.folderId || 'none', parseInt(r.count)])
    );
    
    return { total, archived, favorites, byFolder };
  }
}

export const noteRepository = new NoteRepository();
```

---

## 5. Migrations

### src/main/db/migrations/1700000000000-CreateTables.ts

```typescript
import { MigrationInterface, QueryRunner, Table, Index } from 'typeorm';

export class CreateTables1700000000000 implements MigrationInterface {
  name = 'CreateTables1700000000000';
  
  async up(queryRunner: QueryRunner): Promise<void> {
    // Folders table
    await queryRunner.createTable(new Table({
      name: 'folders',
      columns: [
        { name: 'id', type: 'varchar', isPrimary: true },
        { name: 'name', type: 'varchar', length: '100' },
        { name: 'color', type: 'varchar', length: '7', default: "'#6366f1'" },
        { name: 'parent_id', type: 'varchar', isNullable: true },
        { name: 'created_at', type: 'datetime', default: 'CURRENT_TIMESTAMP' },
      ],
      foreignKeys: [
        {
          columnNames: ['parent_id'],
          referencedTableName: 'folders',
          referencedColumnNames: ['id'],
          onDelete: 'SET NULL',
        },
      ],
    }));
    
    // Notes table
    await queryRunner.createTable(new Table({
      name: 'notes',
      columns: [
        { name: 'id', type: 'varchar', isPrimary: true },
        { name: 'title', type: 'varchar', length: '255' },
        { name: 'content', type: 'text', isNullable: true },
        { name: 'archived', type: 'boolean', default: 0 },
        { name: 'favorite', type: 'boolean', default: 0 },
        { name: 'tags', type: 'json', isNullable: true },
        { name: 'folder_id', type: 'varchar', isNullable: true },
        { name: 'created_at', type: 'datetime', default: 'CURRENT_TIMESTAMP' },
        { name: 'updated_at', type: 'datetime', default: 'CURRENT_TIMESTAMP' },
      ],
      foreignKeys: [
        {
          columnNames: ['folder_id'],
          referencedTableName: 'folders',
          referencedColumnNames: ['id'],
          onDelete: 'SET NULL',
        },
      ],
    }));
    
    // Index
    await queryRunner.createIndex('notes', new Index({
      name: 'IDX_NOTES_TITLE',
      columnNames: ['title'],
    }));
    
    await queryRunner.createIndex('notes', new Index({
      name: 'IDX_NOTES_FOLDER',
      columnNames: ['folder_id'],
    }));
    
    // FTS5 table สำหรับ full-text search
    await queryRunner.query(`
      CREATE VIRTUAL TABLE notes_fts USING fts5(
        title, content,
        content=notes,
        content_rowid=rowid
      )
    `);
    
    // Triggers สำหรับ sync FTS
    await queryRunner.query(`
      CREATE TRIGGER notes_fts_insert AFTER INSERT ON notes BEGIN
        INSERT INTO notes_fts(rowid, title, content) VALUES (new.rowid, new.title, new.content);
      END
    `);
    
    await queryRunner.query(`
      CREATE TRIGGER notes_fts_update AFTER UPDATE ON notes BEGIN
        UPDATE notes_fts SET title=new.title, content=new.content WHERE rowid=old.rowid;
      END
    `);
    
    await queryRunner.query(`
      CREATE TRIGGER notes_fts_delete AFTER DELETE ON notes BEGIN
        DELETE FROM notes_fts WHERE rowid=old.rowid;
      END
    `);
  }
  
  async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query('DROP TRIGGER IF EXISTS notes_fts_delete');
    await queryRunner.query('DROP TRIGGER IF EXISTS notes_fts_update');
    await queryRunner.query('DROP TRIGGER IF EXISTS notes_fts_insert');
    await queryRunner.query('DROP TABLE IF EXISTS notes_fts');
    await queryRunner.dropTable('notes', true);
    await queryRunner.dropTable('folders', true);
  }
}
```

---

## 6. Transactions

```typescript
// src/main/db/services/NoteService.ts
import { AppDataSource } from '../database';
import { Note, Folder } from '../entities/Note';

export class NoteService {
  // Transaction ตัวอย่าง
  async moveNotesToFolder(noteIds: string[], folderId: string): Promise<void> {
    await AppDataSource.transaction(async (manager) => {
      // ตรวจสอบว่า folder มีอยู่
      const folder = await manager.findOneOrFail(Folder, { where: { id: folderId } });
      
      // อัพเดท notes ทั้งหมด
      await manager
        .createQueryBuilder()
        .update(Note)
        .set({ folderId: folder.id })
        .whereInIds(noteIds)
        .execute();
    });
  }
  
  // Nested transaction
  async duplicateFolder(folderId: string): Promise<Folder> {
    return AppDataSource.transaction(async (manager) => {
      const original = await manager.findOneOrFail(Folder, {
        where: { id: folderId },
        relations: ['notes'],
      });
      
      // สร้าง folder ใหม่
      const newFolder = manager.create(Folder, {
        name: `${original.name} (Copy)`,
        color: original.color,
      });
      await manager.save(newFolder);
      
      // Copy notes
      for (const note of original.notes) {
        const newNote = manager.create(Note, {
          title: note.title,
          content: note.content,
          tags: note.tags,
          folderId: newFolder.id,
        });
        await manager.save(newNote);
      }
      
      return newFolder;
    });
  }
}
```

---

## 7. Prisma กับ Electron (Workaround)

```bash
npm install prisma @prisma/client
npm install --save-dev prisma
```

### prisma/schema.prisma

```prisma
generator client {
  provider = "prisma-client-js"
  binaryTargets = ["native", "linux-arm64-openssl-1.1.x"]
  // ต้องระบุ output เพื่อให้ work กับ Electron
  output = "../node_modules/.prisma/client"
}

datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

model Note {
  id        String   @id @default(uuid())
  title     String
  content   String?
  archived  Boolean  @default(false)
  favorite  Boolean  @default(false)
  tags      String?  // JSON string
  folderId  String?
  folder    Folder?  @relation(fields: [folderId], references: [id])
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Folder {
  id    String  @id @default(uuid())
  name  String
  color String  @default("#6366f1")
  notes Note[]
}
```

### ตั้งค่า Prisma สำหรับ Electron

```typescript
// src/main/db/prisma-client.ts
import { PrismaClient } from '@prisma/client';
import { join } from 'path';
import { app } from 'electron';

// ต้องตั้งค่า DATABASE_URL ก่อน require PrismaClient
const dbPath = join(app.getPath('userData'), 'prisma.db');
process.env.DATABASE_URL = `file:${dbPath}`;

export const prisma = new PrismaClient({
  log: process.env.NODE_ENV === 'development' 
    ? ['query', 'info', 'warn', 'error'] 
    : ['error'],
});

export async function connectPrisma(): Promise<void> {
  await prisma.$connect();
}

export async function disconnectPrisma(): Promise<void> {
  await prisma.$disconnect();
}
```

---

## 8. สรุป

| ORM | ข้อดี | ข้อเสีย |
|-----|-------|---------|
| TypeORM | Decorator-based, flexible | Config ซับซ้อน |
| Prisma | Type-safe, migrations ดี | ต้องมี workaround สำหรับ Electron |
| Sequelize | ใช้ง่าย, mature | TypeScript support ไม่ดีเท่า |
| Knex | Query builder, flexible | ต้องเขียน SQL มากกว่า |
| better-sqlite3 | เร็วที่สุด, synchronous | ต้องเขียน SQL เอง |

### Checklist

- [ ] ใช้ Migrations แทน synchronize
- [ ] ตั้งค่า WAL mode สำหรับ SQLite performance
- [ ] Rebuild native modules หลังเปลี่ยน Electron version
- [ ] Backup database ก่อน migrate
- [ ] Test migrations ทั้ง up และ down

---

*จบ Part 054 - ต่อไป Part 055: System Notifications Advanced*
