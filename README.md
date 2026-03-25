# CommitLens

Absolutely! Let’s make this beginner-friendly, with simpler explanations, more guidance, and exact commands you can copy-paste, so even if you’re new to web development, you can follow along and build CommitLens in one week.

CommitLens: Beginner-Friendly 1-Week Tutorial

Goal: Build a simple SaaS for developers to analyze GitHub commits, using Next.js + React, NestJS, Supabase, and Stripe.

Timeframe: 7 days (1 week)

Day 1 – Set Up Your Projects
1. Set up the Frontend (Next.js + React + Tailwind)
Open VS Code → Terminal → Run:
mkdir commitlens-web
cd commitlens-web
npx create-next-app@latest .
Choose TypeScript → Yes
Choose App Router → Yes
Install Tailwind CSS (for styling):
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
Configure Tailwind (tailwind.config.js):
module.exports = {
  content: ["./app/**/*.{js,ts,jsx,tsx}", "./components/**/*.{js,ts,jsx,tsx}"],
  theme: { extend: {} },
  plugins: [],
}
Add Tailwind to CSS (styles/globals.css):
@tailwind base;
@tailwind components;
@tailwind utilities;
2. Set up the Backend (NestJS)
Open a new terminal in VS Code:
mkdir commitlens-api
cd commitlens-api
npm i -g @nestjs/cli
nest new .
Choose npm
Install dependencies:
npm install @nestjs/config @nestjs/passport passport passport-github2 stripe @supabase/supabase-js
npm install --save-dev @types/passport-github2
3. Create Supabase Project
Go to Supabase
 → New Project
Create 3 tables:

Users Table

Column	Type
id	uuid (primary key)
email	text
github_username	text
stripe_customer_id	text

Commit Reports Table

Column	Type
id	uuid (primary key)
user_id	uuid (foreign key → users)
repo	text
score	int
feedback	text
recruiter_notes	text
created_at	timestamp

Subscriptions Table

Column	Type
user_id	uuid (foreign key → users)
status	text
plan	text
4. Stripe Setup
Create a Stripe account → Products → Prices
Save your secret key and publishable key (we’ll use them later)
5. Create Folder Structure

Frontend

commitlens-web/
├─ app/
│  ├─ dashboard/page.tsx
│  ├─ analyze/page.tsx
│  ├─ auth/login.tsx
│  └─ billing/page.tsx
├─ components/
│  ├─ Navbar.tsx
│  └─ CommitCard.tsx
├─ lib/
│  └─ supabaseClient.ts
├─ styles/globals.css

Backend

commitlens-api/
├─ src/
│  ├─ auth/
│  │  ├─ auth.module.ts
│  │  └─ github.strategy.ts
│  ├─ repos/
│  │  ├─ repos.module.ts
│  │  ├─ repos.service.ts
│  │  └─ repos.controller.ts
│  ├─ billing/
│  │  ├─ billing.module.ts
│  │  └─ billing.controller.ts
│  ├─ main.ts
│  └─ app.module.ts
Day 2 – Authentication and Billing
1. Connect Supabase Auth (GitHub Login)
In Supabase → Authentication → Settings → External OAuth Providers → GitHub
Add .env in commitlens-web:
NEXT_PUBLIC_SUPABASE_URL=<your-supabase-url>
NEXT_PUBLIC_SUPABASE_ANON_KEY=<anon-key>
SUPABASE_SERVICE_KEY=<service-key>
Create supabaseClient.ts:
import { createClient } from '@supabase/supabase-js';

export const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
);
Add GitHub login button (app/auth/login.tsx):
'use client';
import { supabase } from '../../lib/supabaseClient';

export default function Login() {
  const handleLogin = async () => {
    await supabase.auth.signInWithOAuth({ provider: 'github' });
  };
  return <button onClick={handleLogin}>Login with GitHub</button>;
}
2. Stripe Checkout
Frontend /billing/page.tsx:
'use client';
import { loadStripe } from '@stripe/stripe-js';

const stripePromise = loadStripe(process.env.NEXT_PUBLIC_STRIPE_KEY!);

export default function Billing() {
  const handleCheckout = async () => {
    const res = await fetch('/api/create-checkout-session', { method: 'POST' });
    const { sessionId } = await res.json();
    const stripe = await stripePromise;
    await stripe?.redirectToCheckout({ sessionId });
  };
  return <button onClick={handleCheckout}>Subscribe</button>;
}
Backend /billing/billing.controller.ts:
import { Controller, Post, Body } from '@nestjs/common';
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: '2022-11-15' });

@Controller('billing')
export class BillingController {
  @Post('create-checkout-session')
  async createCheckout(@Body() body: { customerId: string }) {
    const session = await stripe.checkout.sessions.create({
      customer: body.customerId,
      payment_method_types: ['card'],
      line_items: [{ price: 'price_id', quantity: 1 }],
      mode: 'subscription',
      success_url: 'https://yourdomain.com/success',
      cancel_url: 'https://yourdomain.com/cancel',
    });
    return { sessionId: session.id };
  }
}
Day 3 – Fetch GitHub Commits
Use GitHub API:
// backend/repos.service.ts
import fetch from 'node-fetch';

export class ReposService {
  async getCommits(token: string, repo: string) {
    const res = await fetch(`https://api.github.com/repos/${repo}/commits`, {
      headers: { Authorization: `token ${token}` },
    });
    return res.json();
  }
}
Save commits to Supabase:
await supabase.from('commit_reports').insert([
  { user_id: userId, repo: repoName, score: null, feedback: null },
]);
Day 4 – AI Analysis of Commits
Install OpenAI SDK:
npm install openai
Analyze commits:
import OpenAI from 'openai';
const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

export async function analyzeCommits(commitMessages: string[]) {
  const response = await openai.chat.completions.create({
    model: 'gpt-4',
    messages: [{ role: 'user', content: `Analyze these commits: ${commitMessages.join('\n')}` }],
  });
  const text = response.choices[0].message?.content;
  return { score: 80, feedback: text, recruiterNotes: 'Looks professional' };
}
Save analysis to Supabase:
await supabase.from('commit_reports').update({
  score: analysis.score,
  feedback: analysis.feedback,
  recruiter_notes: analysis.recruiterNotes
}).eq('id', reportId);
Day 5 – Dashboard UI
Dashboard (app/dashboard/page.tsx):
'use client';
import { useEffect, useState } from 'react';

export default function Dashboard() {
  const [reports, setReports] = useState([]);

  useEffect(() => {
    fetch('/api/reports').then(res => res.json()).then(setReports);
  }, []);

  return (
    <div className="p-4">
      {reports.map(r => (
        <div key={r.id} className="bg-gray-900 text-green-400 p-4 rounded-lg font-mono">
          Repo: {r.repo} | Score: {r.score}
        </div>
      ))}
    </div>
  );
}
Analyze button (app/analyze/page.tsx):
<button onClick={() => fetch('/api/analyze', { method: 'POST' })}>Analyze Repo</button>
Day 6 – Stripe Webhooks & PDF Export
Stripe webhook:
@Post('webhook')
async webhook(@Req() req) {
  const event = stripe.webhooks.constructEvent(
    req.body, req.headers['stripe-signature'], process.env.STRIPE_WEBHOOK_SECRET!
  );
  if (event.type === 'checkout.session.completed') {
    // update subscription in Supabase
  }
}
Generate PDF (optional MVP):
import PDFDocument from 'pdfkit';
const doc = new PDFDocument();
doc.text(`Commit Report: ${repoName}\nScore: ${score}\nFeedback: ${feedback}`);
doc.pipe(fs.createWriteStream('report.pdf'));
doc.end();
Day 7 – Launch & Marketing
Landing page: Simple pitch → GitHub login → “Get Started”
Deploy:
Frontend: Vercel
Backend: Railway or Fly.io
Set environment variables (.env) in deployment
Marketing: Hacker News, Reddit r/cscareerquestions, Twitter/X
Offer a free trial → Encourage early subscribers
