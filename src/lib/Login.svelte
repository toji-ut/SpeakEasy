<script lang="ts">
  import { currentUser, pb } from './pocketbase';

  let username: string;
  let password: string;
  let errorMessage: string = '';

  async function login() {
    if (!username || !password) {
      errorMessage = 'Please enter both a username and password.';
      return;
    }

    try {
      const user = await pb.collection('users').authWithPassword(username, password);
      console.log(user);
    } catch (err) {
      errorMessage = 'Incorrect username or password.';
    }
  }

  async function signUp() {
    if (!username || !password) {
      errorMessage = 'Please enter both a username and password.';
      return;
    }

    try {
      const data = {
        username,
        password,
        passwordConfirm: password,
        name: username,
      };
      const createdUser = await pb.collection('users').create(data);
      await login();
    } catch (err) {
      errorMessage = 'Error signing up. Please try again.';
    }
  }

  function signOut() {
    pb.authStore.clear();
    errorMessage = '';
  }
</script>

{#if $currentUser}
  <p>
    Signed in as -> {$currentUser.username}
    <button on:click={signOut}>Sign Out</button>
  </p>
{:else}
  <form on:submit|preventDefault>
    <div style="display: flex; flex-direction: column; width: 266px;">
      <input
        placeholder="Username"
        type="text"
        bind:value={username}
        style="width: 100%; padding: 10px; margin-bottom: 10px;"
      />
      <input 
        placeholder="Password" 
        type="password" 
        bind:value={password} 
        style="width: 100%; padding: 10px; margin-bottom: 10px;"
      />
    </div>
    <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px;">
      <div style="width: 145px;">
        <button on:click={signUp} style="width: 100%;">Sign Up</button>
      </div>
      <div style="width: 145px;">
        <button on:click={login} style="width: 100%;">Login</button>
      </div>
    </div>
    <div style="display: flex; justify-content: space-between; align-items: center;">
      <p style="color: red; width: 190px; text-align: left;">{errorMessage}</p>
    </div>
  </form>
{/if}
