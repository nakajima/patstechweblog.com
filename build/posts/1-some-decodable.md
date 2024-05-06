---
title: `SomeDecodable`
author: Pat Nakajima
publishedAt: 05/04/2024
excerpt:
tags: codable, json, good or bad idea?, swift
---
<p>Sometimes you want to parse JSON that doesn’t really agree with Codable’s synthesized init. Like if a key wants to be a single thing <strong>or</strong> an array of that thing. What am I talking about? Ok let’s say you want to parse some JSON into this:</p><pre><code><span class="keyword">struct</span> BlogPost: <span class="type">Codable</span> {
  <span class="keyword">let</span> author: <span class="type">String</span>
}
</code></pre><p>So the JSON you get can look like this:</p><pre><code>{
  <span class="string">"author"</span>: <span class="string">"pat"</span>
}
</code></pre><p>But <em>TWIST</em>, sometimes it gives you this:</p><pre><code>{
  <span class="string">"author"</span>: [<span class="string">"pat"</span>, <span class="string">"evil pat"</span>]
}
</code></pre><p>With <code>Codable</code> that usually means you have to write your own <code>init(from decoder: any Decoder)</code>. And for me that means I usually have to google how to do that again.</p><p>Well now in this very specific case, I don’t anymore because I’m using this:</p><pre><code><span class="keyword">public enum</span> SomeDecodable&lt;T: <span class="type">Decodable</span>&gt;: <span class="type">Decodable</span> {
	<span class="keyword">case</span> none, <span class="call">one</span>(<span class="type">T</span>), <span class="call">many</span>([<span class="type">T</span>])

	<span class="keyword">public init</span>(from decoder: <span class="keyword">any</span> <span class="type">Decoder</span>) <span class="keyword">throws</span> {
		<span class="keyword">let</span> container = <span class="keyword">try</span>! decoder.<span class="call">singleValueContainer</span>()

		<span class="keyword">if let</span> one = <span class="keyword">try</span>? container.<span class="call">decode</span>(<span class="type">T</span>.<span class="keyword">self</span>) {
			<span class="keyword">self</span> = .<span class="call">one</span>(one)
		} <span class="keyword">else if let</span> many = <span class="keyword">try</span>? container.<span class="call">decode</span>([<span class="type">T</span>].<span class="keyword">self</span>) {
			<span class="keyword">self</span> = .<span class="call">many</span>(many)
		} <span class="keyword">else</span> {
			<span class="keyword">self</span> = .<span class="dotAccess">none</span>
		}
	}
}

<span class="comment">// SomeDecodable can have some Equatable, as a treat.</span>
<span class="keyword">extension</span> <span class="type">SomeDecodable</span>: <span class="type">Equatable</span> <span class="keyword">where</span> <span class="type">T</span>: <span class="type">Equatable</span> { }
</code></pre><p>Now, in our model, we can use it:</p><pre><code><span class="keyword">struct</span> BlogPost {
  <span class="keyword">let</span> author: <span class="type">SomeDecodable</span>&lt;<span class="type">String</span>&gt;
}

<span class="keyword">let</span> post = <span class="keyword">try</span> <span class="type">JSONDecoder</span>().<span class="call">decode</span>(<span class="type">BlogPost</span>.<span class="keyword">self</span>, from: json)
<span class="keyword">switch</span> post.<span class="property">author</span> {
<span class="keyword">case let</span> .<span class="call">one</span>(author):
  <span class="call">print</span>(<span class="string">"The author is named</span> \(author)<span class="string">"</span>)
<span class="keyword">case let</span> .<span class="call">many</span>(authors):
  <span class="call">print</span>(<span class="string">"The authors are named</span> \(authors.<span class="call">joined</span>(separator: <span class="string">"</span>, <span class="string">"</span>))<span class="string">"</span>)
<span class="keyword">default</span>:
  <span class="call">print</span>(<span class="string">"I have no idea who the authors are"</span>)
}
</code></pre><p>With <a href="https://www.swift.org/blog/pack-iteration/">parameter pack iteration</a> in Swift 6.0, we might be able to get rid of the enum and support more generic cases, but I haven’t figured out how to do it with Swift 5.10 yet.</p>