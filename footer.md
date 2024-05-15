Hi I’m Pat. This is my weblog. You can say hi on <a href="https://mastodon.social/@fancypat" rel="me">Mastodon</a>. I’m also on
[GitHub](https://github.com/nakajima).

The markdown lives [here](https://github.com/nakajima/patstechweblog.com) and I generate
the HTML with [this](https://github.com/nakajima/ohno).

<script>
  window.addEventListener('hashchange', function() {
    for (const footnote of document.querySelectorAll('.current')) {
      footnote.classList.remove('current')
    }

    if (/footnote-link-\d/.test(document.location.hash)) {
      // Remove footnote hash when going back
      history.replaceState("", document.title, window.location.pathname + window.location.search)
    } else if (/footnote-\d/.test(document.location.hash)) {
      // Highlight current footnote
      const highlightedFootnote = document.querySelector(document.location.hash)
      if (highlightedFootnote) {
        highlightedFootnote.classList.add('current')
      }
    }
  })
</script>
