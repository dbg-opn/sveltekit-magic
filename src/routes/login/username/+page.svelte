<script lang="ts">
  import AuthCheck from "$lib/components/AuthCheck.svelte";
  import { db, user, userData } from "$lib/firebase";
  import { doc, getDoc, writeBatch } from "firebase/firestore";

  let username = "";
  let loading = false;
  let isAvailable = false;

  let debounceTimer: NodeJS.Timeout;

  const re = /^(?=[a-zA-Z0-9._]{3,16}$)(?!.*[_.]{2})[^_.].*[^_.]$/;

  // Reactive declaration, changes every time the value in user input in form changes
  $: isValid =
    username?.length > 2 && username.length < 16 && re.test(username);
  $: isTouched = username.length > 0;
  // Can be combined, can create bugs hard to track down,
  // if you have multiple states depending on eachother
  $: isTaken = isValid && !isAvailable && !loading;

  async function checkAvailability() {
    isAvailable = false;
    clearTimeout(debounceTimer);

    loading = true;

    // Prevents firing a db query with each key stroke
    debounceTimer = setTimeout(async () => {
      const ref = doc(db, "usernames", username);
      const exists = await getDoc(ref).then((doc) => doc.exists());

      isAvailable = !exists;
      loading = false;
    }, 500);
  }

  async function confirmUsername() {
    const batch = writeBatch(db);
    batch.set(doc(db, "usernames", username), { uid: $user?.uid });
    batch.set(doc(db, "users", $user!.uid), {
      username,
      photoURL: $user?.photoURL ?? null,
      published: true,
      bio: "I am the Walrus",
      links: [
        {
          title: "Test Link",
          url: "https://kung.foo",
          icon: "custom",
        },
      ],
    });
    await batch.commit();
  }
</script>

<AuthCheck>
  {#if $userData?.username}
    <p class="text-lg">
      Your username is <span class="text-success font-bold"
        >@{$userData.username}</span
      >
    </p>
    <p class="text-sm">(Usernames cannot be changed)</p>
    <a href="/login/photo" class="btn btn-primary">Upload Profile Image</a>
  {:else}
    <form action="" class="w-2/5" on:submit|preventDefault={confirmUsername}>
      <input
        type="text"
        placeholder="Username"
        class="input w-full"
        bind:value={username}
        on:input={checkAvailability}
        class:input-error={!isValid && isTouched}
        class:input-warning={isTaken}
        class:input-success={isAvailable && isValid && !loading}
      />

      <div class="my-4 min-h-16 px-8 w-full">
        {#if loading}
          <p class="text-secondary">
            Checking availability of @{username}...
          </p>{/if}
        {#if !isValid && isTouched}
          <p class="text-error text-sm">
            must be 3-16 characters long, alphanumeric only
          </p>
        {/if}

        {#if isValid && !isAvailable && !loading}
          <p class="text-warning text-sm">@{username} is not available</p>
        {/if}

        {#if isAvailable}
          <button class="btn btn-success">Confirm username @{username}</button>
        {/if}
      </div>
    </form>
  {/if}
</AuthCheck>
