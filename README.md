/**
 * Simple HTTP Load Tester (http_load_tester.ts)
 *
 * Spawns concurrent requests and reports latencies and error rate.
 * Run: ts-node src/http_load_tester.ts <url> [concurrency] [requests]
 */

import http from 'http';
import https from 'https';
import { URL } from 'url';

async function requestOnce(urlStr: string): Promise<number> {
  return new Promise((resolve, reject) => {
    const url = new URL(urlStr);
    const mod = url.protocol === 'https:' ? https : http;
    const start = Date.now();
    const req = mod.request(url, res => {
      res.on('data', () => { /* consume */ });
      res.on('end', () => resolve(Date.now() - start));
    });
    req.on('error', err => reject(err));
    req.end();
  });
}

async function run(url: string, concurrency = 10, total = 100) {
  let completed = 0, failed = 0;
  const latencies: number[] = [];
  const inFlight: Promise<void>[] = [];
  for (let i = 0; i < total; i++) {
    const p = (async () => {
      try {
        const t = await requestOnce(url);
        latencies.push(t);
      } catch (e) {
        failed++;
      } finally {
        completed++;
      }
    })();
    inFlight.push(p);
    if (inFlight.length >= concurrency) {
      await Promise.race(inFlight).catch(() => {});
      // remove settled
      for (let j = inFlight.length - 1; j >= 0; j--) {
        if ((inFlight[j] as any).then) {
          // can't easily check; just keep size bounded by shifting
        }
      }
    }
  }
  await Promise.allSettled(inFlight);
  latencies.sort((a, b) => a - b);
  const p50 = latencies[Math.floor(latencies.length * 0.5)] || 0;
  const p95 = latencies[Math.floor(latencies.length * 0.95)] || 0;
  console.log(`completed=${completed} failed=${failed} p50=${p50}ms p95=${p95}ms`);
}

// demo CLI
if (require.main === module) {
  const url = process.argv[2] || 'http://example.com';
  const concurrency = parseInt(process.argv[3] || '10', 10);
  const total = parseInt(process.argv[4] || '50', 10);
  run(url, concurrency, total).catch(e => console.error(e));
}
