<!-- Messages.svelte -->
<script lang="ts">
  import { onMount, onDestroy, tick } from 'svelte';
  import { currentUser, pb } from './pocketbase';

  let newMessage: string;
  let messages = {};
  let messageOrder = [];
  let messagesContainer;
  let unsubscribe: () => void;

  const emojiCountFields = {
    '😃': 'smile_count',
    '🖤': 'heart_count',
    '💩': 'poop_count',
    '🔥': 'fire_count',
  };

  // Initialize user reactions from local storage
  let userReactions = {};

  onMount(async () => {
    // Load emoji counts and initial messages when the component mounts
    await loadEmojiCounts();
    await loadInitialMessages();

    // Load user reactions from local storage
    const storedReactions = localStorage.getItem('userReactions');
    if (storedReactions) {
      userReactions = JSON.parse(storedReactions);
    }

    // Subscribe to real-time messages
    unsubscribe = await pb.collection('messages').subscribe('*', async ({ action, record }) => {
      if (action === 'create') {
        const user = await pb.collection('users').getOne(record.user);
        record.expand = { user };
        messages[record.id] = record; // Add new message to the dictionary
        messageOrder.push(record.id); // Append the message ID to the order array
        await loadEmojiCounts(); // Update emoji counts
        await tick(); // Ensure the new message is rendered
        scrollToBottom(); // Scroll to the bottom when a new message is added
      }
      if (action === 'delete') {
        delete messages[record.id];
        messageOrder = messageOrder.filter(id => id !== record.id);
      }
    });
  });

  onMount(() => {
    // Scroll to the bottom when the page is refreshed
    scrollToBottom();
  });

  onDestroy(() => {
    unsubscribe?.();
  });

  // Scroll to the bottom of the messages container
  function scrollToBottom() {
    if (messagesContainer) {
      messagesContainer.scrollTop = messagesContainer.scrollHeight;
    }
  }

  // Function to check if the current user has reacted with a specific emoji for a message
  function hasUserReacted(message, emoji) {
    return userReactions[message.id] && userReactions[message.id][emoji] === $currentUser.id;
  }

  async function sendMessage() {
    const data = {
      text: newMessage,
      user: $currentUser.id,
    };
    const createdMessage = await pb.collection('messages').create(data);
    newMessage = '';
  }

  function getAvatarSeed(user) {
    return user.username;
  }

  async function loadInitialMessages() {
    const resultList = await pb.collection('messages').getList(1, 50, {
      sort: 'created',
      expand: 'user',
    });

    // Initialize messages and order based on the initial result
    resultList.items.forEach((message) => {
      messages[message.id] = message;
      messageOrder.push(message.id);
    });
  }

  async function loadEmojiCounts() {
    for (let i = 0; i < messageOrder.length; i++) {
      const message = messages[messageOrder[i]];
      for (const emoji in emojiCountFields) {
        const emojiCountField = emojiCountFields[emoji];
        const response = await pb.collection('messages').getOne(message.id);
        if (response) {
          messages[message.id][emojiCountField] = response[emojiCountField];
        }
      }
    }
  }

  // Function to toggle user reactions and update local storage
  async function toggleEmojiReaction(message, emoji) {
    if (!userReactions[message.id]) {
      userReactions[message.id] = {};
    }

    if (!hasUserReacted(message, emoji)) {
      // User hasn't reacted with this emoji, so add their reaction
      userReactions[message.id][emoji] = $currentUser.id;
      const emojiCountField = emojiCountFields[emoji];
      if (emojiCountField) {
        message[emojiCountField] = (message[emojiCountField] || 0) + 1;
        await pb.collection('messages').update(message.id, {
          [emojiCountField]: message[emojiCountField],
        });
      }
    } else {
      // User has already reacted with this emoji, so remove their reaction
      delete userReactions[message.id][emoji];
      const emojiCountField = emojiCountFields[emoji];
      if (emojiCountField) {
        message[emojiCountField] = (message[emojiCountField] || 0) - 1;
        await pb.collection('messages').update(message.id, {
          [emojiCountField]: message[emojiCountField],
        });
      }
    }
    
    // Update local storage with user reactions
    localStorage.setItem('userReactions', JSON.stringify(userReactions));
    await loadEmojiCounts();
  }
</script>

<!-- Svelte HTML Markup -->
<div class="msg-container">
  <div class="msg-feed">
    <div class="messages-container" bind:this={messagesContainer}>
      <div class="messages">
        {#each messageOrder as messageId}
          {#if messages[messageId]} <!-- Check if the message exists -->
            <div class="msg">
              <div class="msg-content">
                <div class="msg-header">
                  <div class="msg-sent-by">
                    <small>
                      Sent by @{messages[messageId].expand?.user?.username}
                    </small>
                  </div>
                  <div class="msg-avatar">
                    <img
                      class="avatar"
                      src={`https://api.dicebear.com/7.x/pixel-art/svg?seed=${getAvatarSeed(messages[messageId].expand?.user)}`}
                      alt="avatar"
                      width="40px"
                    />
                  </div>
                </div>
                <p class="msg-text">{messages[messageId].text}</p>
                <div class="emoji-reactions">
                  {#each ['😃', '🖤', '💩', '🔥'] as emoji}
                    <div class="emoji-button">
                      <button on:click={() => toggleEmojiReaction(messages[messageId], emoji)}>
                        {emoji} <span>{messages[messageId][emojiCountFields[emoji]] || 0}</span>
                      </button>
                    </div>
                  {/each}
                </div>
              </div>
            </div>
          {/if}
        {/each}
      </div>
    </div>
    <form on:submit|preventDefault={sendMessage}>
      <div class="input-container">
        <input placeholder="Send a message" type="text" bind:value={newMessage} class="message-input" />
        <button type="submit" class="send-button">Send</button>
      </div>
    </form>
  </div>
</div>

<style>
  /* Svelte Component Styles (Add to your Svelte component) */

  .messages-container {
    border: 2px solid #ffffff;
    border-radius: 10px;
    max-height: 400px; /* Set a maximum height for the messages container */
    overflow-y: scroll; /* Enable vertical scrolling if messages exceed the container height */
    width: 317px; /* Adjust the width to match the message div width */
    margin-bottom: 5px;
  }

  .msg-container {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    width: 100%;
    max-width: 1200px;
    margin: 0 auto;
  }

  .msg-feed {
    width: 75%;
    /* Add styling for the main feed (e.g., background color, borders, etc.) */
  }

  .messages {
    width: 306px; /* Adjust the width to make the message div a bit smaller */
  }

  .msg {
    display: flex;
    padding: 10px;
    border: 2px solid #ffffff;
    height: auto; /* Allow the div to adjust its height as needed */
    border-radius: 10px; /* Add border radius for smooth corners */
    margin-bottom: 3px;
  }

  .msg-avatar {
    align-self: center; /* Move the avatar to the left center */
  }

  .msg-content {
    width: 100%;
    display: flex;
    flex-direction: column;
  }

  .msg-header {
    display: flex;
    align-items: flex-start; /* Move "Sent by" a bit higher */
  }

  .msg-sent-by {
    flex: 1;
    font-size: 27px;
  }

  .msg-text {
    font-size: 27px; /* Increase the font size */
    font-weight: bold; /* Make the font bold */
    font-style: italic;
    max-height: 200px; /* Adjust the maximum height to make the message div smaller */
    overflow-y: auto; /* Enable vertical scrolling if content exceeds max height */
  }

  .emoji-reactions {
    margin-top: 10px;
    display: flex;
    align-items: center;
  }

  .emoji-button {
    font-size: 18px;
    margin-right: 10px;
    cursor: pointer;
  }

  .emoji-button button {
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  .input-container {
    display: flex;
    justify-content: space-between; /* Move the input and button to the right */
    width: 100%;
  }

  .message-input {
    flex: 1;
    padding: 10px;
    border-radius: 10px 0 0 10px;
    border: 2px solid #ffffff;
    min-width: 237px; /* Set a minimum width for the input */
  }

  .send-button {
    background: #ffffff;
    color: black;
    font-weight: bold;
    border: none;
    padding: 10px;
    cursor: pointer;
    border-radius: 0 5px 5px 0;
  }

  .messages-container::-webkit-scrollbar {
    width: 0.69rem;
  }

  .messages-container::-webkit-scrollbar-track {
    background: transparent; /* Set a transparent background */
  }

  .messages-container::-webkit-scrollbar-thumb {
    background: #d1d1d1; /* Adjust the thumb color to match the Mac scrollbar */
    border-radius: 1rem; /* Add some border radius for a more rounded look */
  }
</style>
