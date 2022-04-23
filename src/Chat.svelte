<script>
    import Login from './Login.svelte';
    import ChatMessage from './ChatMessage.svelte';
    import { username, user} from './user';
    import {onMount} from 'svelte';
    import debounce from 'lodash.debounce';
    import GUN from 'gun';
    const db = GUN();
    let newMessage; // Form input
    let messages = [];  // Message/User/Timestamp
    let scrollBottom;
    let lastScrollTop;
    let canAutoScroll = true;
    let unreadMessages = false;
    function autoScroll() {
        setTimeout(() => scrollBottom?.scrollIntoView({ behavior: 'auto' }), 50);
        unreadMessages = false;
    }
  
    function watchScroll(e) {
        canAutoScroll = (e.target.scrollTop || Infinity) > lastScrollTop;
        lastScrollTop = e.target.scrollTop;
    }
    
    $: debouncedWatchScroll = debounce(watchScroll, 1000);
    onMount(() => {
    var match = {
      // lexical queries are kind of like a limited RegEx or Glob.
      '.': {
        // property selector
        '>': new Date(+new Date() - 1 * 1000 * 60 * 60 * 3).toISOString(), // find any indexed property larger ~3 hours ago
      },
      '-': 1, // filter in reverse
    };
    // Get Messages
    db.get('chat')
      .map(match)
      .once(async (data, id) => { // Use the on method if you want the messages to be updated in real time.
        if (data) {
          // Key for end-to-end encryption
          // Not secure since the key is hardcoded
          const key = '#foo';
          // Formating the message
          var message = {
            who: await db.user(data).get('alias'), // The sender
            what: (await SEA.decrypt(data.what, key)) + '', // The actual message content
            when: GUN.state.is(data, 'what'), // The timestamp
          };
          if (message.what) {
            // Adds the messages to the array.
            messages = [...messages.slice(-100), message].sort((a, b) => a.when - b.when);
            if (canAutoScroll) {
              autoScroll();
            } else {
              unreadMessages = true;
            }
          }
        }
      });
  });
  async function sendMessage() {
    const secret = await SEA.encrypt(newMessage, '#foo');
    const message = user.get('all').set({ what: secret });
    const index = new Date().toISOString();
    db.get('chat').get(index).put(message);
    newMessage = '';
    canAutoScroll = true;
    autoScroll();
  }
</script>
<div class="container">
    {#if $username}
      <main on:scroll={debouncedWatchScroll}>
        {#each messages as message (message.when)}
          <ChatMessage {message} sender={$username} />
        {/each}
        <div class="dummy" bind:this={scrollBottom} />
      </main>
      <form on:submit|preventDefault={sendMessage}>
        <input type="text" placeholder="Type a message..." bind:value={newMessage} maxlength="100" />
        <button type="submit" disabled={!newMessage}>💥</button>
      </form>
      
      {#if !canAutoScroll}
      <div class="scroll-button">
        <button on:click={autoScroll} class:red={unreadMessages}>
          {#if unreadMessages}
            💬
          {/if}
          👇
        </button>
      </div>
     {/if}
    {:else}
      <main>
        <Login />
      </main>
    {/if}
  </div>