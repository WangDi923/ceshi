
## Feature Overview

1. The user uploads a file to Supabase object storage.
2. Store file metadata in the Supabase database.
3. Call the DeepSeek AI interface to generate document summaries.
4. Display the generated summary on the front-end page.

## Step1 - Supabase Object Storage (file upload system)
### 1. Create a Supabase project
 open [Supabase](https://supabase.com) log in  
 click `New Project` 

<img width="1920" height="1019" alt="image" src="https://github.com/user-attachments/assets/9e4d6d50-fe81-4eef-a747-7e2eb7b9c024" />

### 2.  Create a Storage Bucket
 Select from the left menu **Storage**，click `Create bucket`。
   - **Name**: `documents`
   - **Public bucket**: ✅ 
  
<img width="1920" height="1019" alt="image" src="https://github.com/user-attachments/assets/d9d41f72-f0ca-44bb-ae1f-92a8d0d4d35b" />

### 3. Obtain API Key
Navigate to `Project Settings` -> `API`, and copy the following content:  
- `Project URL`
- `anon public key`
<img width="1920" height="1019" alt="image" src="https://github.com/user-attachments/assets/227365fa-9d70-4adf-ae0e-fbfafaba3b1f" />

### 4. Environment variable configuration
Create a `.env.local` file in the project root directory and fill in the following configuration:
```env
SUPABASE_URL=my ProjectURL
SUPABASE_ANON_KEY=my AnonKey
```
### 5. Install dependencies
```bash
npm install @supabase/supabase-js
```
### 6. Create a Supabase client
Creat file `app/lib/supabase.ts`：
```typescript
import { createClient } from '@supabase/supabase-js'
export const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_ANON_KEY!
)
```
### 7. File upload interface
Creat file `app/api/upload/route.ts`：
```typescript
import { NextResponse } from 'next/server'
import { supabase } from '@/app/lib/supabase'
export async function POST(req: Request) {
  const formData = await req.formData()
  const file = formData.get('file') as File
  if (!file) {
    return NextResponse.json({ error: "No file" }, { status: 400 })
  }
  const bytes = await file.arrayBuffer()
  const buffer = Buffer.from(bytes)
  const { error } = await supabase.storage
    .from('documents')
    .upload(`uploads/${Date.now()}-${file.name}`, buffer)
  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 })
  }
  return NextResponse.json({
    message: "Upload successful"
  })
}
```

### 8. Create a data table (SQL)
Click **SQL Editor** on the left side of Supabase, click `New Query`, and run the following SQL statement:
```sql
create table public.summaries (
  id uuid default gen_random_uuid() primary key,
  file_name text not null,
  file_path text not null,
  summary text,
  created_at timestamp with time zone default timezone('utc'::text, now()) not null
);
```
<img width="1920" height="1019" alt="image" src="https://github.com/user-attachments/assets/eb3f0464-ba5c-4af3-ba05-e4538d0f0c81" />

## Step2 - Configure DeepSeek AI key
<img width="1910" height="915" alt="image" src="https://github.com/user-attachments/assets/a1ae6ef6-f68e-4759-80e2-89cb252b343c" />  

Update the local environment variables by opening the `my-app/.env.local` file:  
```env
SUPABASE_URL=my ProjectURL
SUPABASE_ANON_KEY=my AnonKey
DEEPSEEK_API_KEY=my DeepSeekAPIKey
```

### 1. AI Summary API
We need a backend interface to receive document content and invoke AI to generate summaries.  
Create file `app/api/summarize/route.ts`：
```typescript
import { NextResponse } from 'next/server';
export async function POST(req: Request) {
  const { text } = await req.json();
  const apiKey = process.env.DEEPSEEK_API_KEY;
  if (!text) return NextResponse.json({ error: "No text provided" }, { status: 400 });
  const response = await fetch("https://api.deepseek.com/chat/completions", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${apiKey}`
    },
    body: JSON.stringify({
      model: "deepseek-chat",
      messages: [
        { role: "system", content: "你是一个专业的文档摘要助手，请用简洁的语言总结用户提供的文本。" },
        { role: "user", content: text }
      ]
    })
  });
  const data = await response.json();
  return NextResponse.json({ summary: data.choices[0].message.content });
}
```

### 2. Improve the front-end UI (integrate upload and summary display)
Replace the content of `app/page.tsx`:
```typescript
'use client'
import { useState } from "react";
export default function Home() {
  const [file, setFile] = useState<File | null>(null);
  const [summary, setSummary] = useState("");
  const [loading, setLoading] = useState(false);
  const handleUpload = async () => {
    if (!file) return;
    setLoading(true);
    
    // 1. 上传到 Supabase
    const formData = new FormData();
    formData.append('file', file);
    await fetch('/api/upload', { method: 'POST', body: formData });
    // 2. 模拟从文件读取文本并生成摘要 (简化版演示)
    const res = await fetch('/api/summarize', {
      method: 'POST',
      body: JSON.stringify({ text: `关于 ${file.name} 的示例文档内容...` })
    });
    const data = await res.json();
    setSummary(data.summary);
    setLoading(false);
  };
  return (
    <div className="p-10 max-w-3xl mx-auto font-sans">
      <h1 className="text-3xl font-bold mb-6 text-slate-800">AI Document Summarizer</h1>
      
      <div className="border-2 border-dashed border-slate-300 p-6 rounded-lg mb-6">
        <input type="file" onChange={(e) => setFile(e.target.files?.[0] || null)} className="mb-4 block" />
        <button 
          onClick={handleUpload}
          disabled={loading || !file}
          className="bg-blue-600 text-white px-6 py-2 rounded disabled:bg-slate-400 hover:bg-blue-700 transition"
        >
          {loading ? "Generating Summary..." : "Upload & Summarize"}
        </button>
      </div>
      {summary && (
        <div className="bg-slate-50 p-6 rounded-lg border border-slate-200">
          <h2 className="text-xl font-semibold mb-3">AI Summary:</h2>
          <p className="text-slate-700 leading-relaxed">{summary}</p>
        </div>
      )}
    </div>
  );
}
```

### Run the project
```bash
npm run dev
```
<img width="1920" height="1019" alt="image" src="https://github.com/user-attachments/assets/f3237f21-f64b-4bbc-9a8e-d2d063f7eaae" />

## Step3 - Beautify the UI and separate functions
The code changes are as follows:  
my-app/app/page.tsx  
my-app/app/api/upload/route.ts  
my-app/app/api/summarize/route.ts  
my-app/app/lib/supabase.ts  
These codes utilize a more refined card layout and separate "upload" and "summary" into two independent actions.  
<img width="1267" height="673" alt="image" src="https://github.com/user-attachments/assets/5941e1a4-2499-4b64-9795-72d0f6a7c6eb" />



## Step4 - Deployment
<img width="1920" height="1019" alt="image" src="https://github.com/user-attachments/assets/fbae7b79-97fa-493a-8a46-c574afc7fdfa" />  
<img width="1920" height="869" alt="image" src="https://github.com/user-attachments/assets/a59fd6c3-3ebf-4dd4-b469-373089b6d1c8" />  
Turn off authentication  
<img width="1920" height="869" alt="image" src="https://github.com/user-attachments/assets/b2e28e58-1026-4a6a-bce2-9582c87d01f2" />  

URL of App:https://platform.deepseek.com/api_keys



