//
// EdgeCache.swift
// Small reverse-proxy like HTTP caching layer (toy demo)
// Uses URLSession to fetch upstream, caches responses in memory with TTL.
// Swift 5+
//

import Foundation

struct CachedResponse {
    let data: Data
    let headers: [String:String]
    let expiry: Date
}

final class EdgeCache {
    private var store: [URL: CachedResponse] = [:]
    private let queue = DispatchQueue(label: "edgecache.sync")
    private let ttlSeconds: TimeInterval
    
    init(ttl: TimeInterval = 30.0) {
        self.ttlSeconds = ttl
    }
    
    func fetch(url: URL, completion: @escaping (Data?, Error?) -> Void) {
        queue.async {
            if let entry = self.store[url], entry.expiry > Date() {
                // Cache hit
                print("[EdgeCache] HIT", url.absoluteString)
                completion(entry.data, nil)
                return
            } else {
                print("[EdgeCache] MISS", url.absoluteString)
                // Cache miss: fetch
                let task = URLSession.shared.dataTask(with: url) { data, resp, err in
                    if let d = data {
                        let headers: [String:String] = ["content-length": "\(d.count)"]
                        let entry = CachedResponse(data: d, headers: headers, expiry: Date().addingTimeInterval(self.ttlSeconds))
                        self.queue.async { self.store[url] = entry }
                        completion(d, nil)
                    } else {
                        completion(nil, err)
                    }
                }
                task.resume()
            }
        }
    }
    
    func invalidate(url: URL) {
        queue.async { self.store.removeValue(forKey: url) }
    }
    
    func stats() -> (count: Int, keys: [String]) {
        queue.sync {
            let keys = Array(store.keys).map { $0.absoluteString }
            return (store.count, keys)
        }
    }
}

// Demo runner: fetch a public endpoint multiple times
let edge = EdgeCache(ttl: 15.0)
let urls = [
    URL(string: "https://httpbin.org/get")!,
    URL(string: "https://httpbin.org/uuid")!
]

let sem = DispatchSemaphore(value: 0)
var pending = urls.count * 3
for u in urls {
    for i in 0..<3 {
        edge.fetch(url: u) { data, err in
            if let d = data {
                print("Fetched \(u.lastPathComponent) size:", d.count)
            } else {
                print("Fetch error:", err ?? "unknown")
            }
            pending -= 1
            if pending == 0 { sem.signal() }
        }
        // slight delay between requests to show caching
        Thread.sleep(forTimeInterval: 1.0)
    }
}
_ = sem.wait(timeout: .now().addingTimeInterval(30))
print("Cache stats:", edge.stats())
