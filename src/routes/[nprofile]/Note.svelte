<script lang="ts">
  import sanitizeHtml from 'sanitize-html';
  import { noteEvents, profileEvents } from '$lib/store';
  import {
    Avatar,
    modalStore,
    type ModalComponent,
  } from '@skeletonlabs/skeleton';
  import { type Event, nip19 } from 'nostr-tools';
  import ModalImage from './ModalImage.svelte';
  import ModalCopyPubkey from './ModalCopyPubkey.svelte';

  export let tag: string[] = [];
  let eventId = '';
  let note: Event | undefined;
  let profile: Event | undefined;
  let content: { display_name: any; picture: any; name: string };

  const allowedTags = ['marquee'];
  const allowedAttributes = {};

  const modalComponent: ModalComponent = {
    // Pass a reference to your custom component
    ref: ModalImage,
    // Add the component properties as key/value pairs
    props: { background: 'bg-red-500' },
    // Provide a template literal for the default component slot
    slot: '<p>Skeleton</p>',
  };

  const pubkeyModalComponent: ModalComponent = {
    // Pass a reference to your custom component
    ref: ModalCopyPubkey,
    // Add the component properties as key/value pairs
    props: { background: 'bg-red-500' },
    // Provide a template literal for the default component slot
    slot: '<p>Skeleton</p>',
  };

  $: if (tag.length > 0) {
    eventId = tag[1];
    note = $noteEvents.find((event) => event.id === eventId);
  }
  $: profile = $profileEvents.find((event) => event.pubkey === note?.pubkey);
  $: if (profile?.content) {
    content = JSON.parse(profile?.content);
  }

  // URL/Image判定の正規表現
  const urlRegex = /(https?:\/\/[^\s]+)/g;

  let emojis = new Map();
  // 絵文字のマッピングを保持するMap
  $: if (note?.tags.length != 0) {
    emojis = new Map(
      note?.tags
        .filter(
          ([tagName, tagContent, url]) =>
            tagName === 'emoji' &&
            tagContent !== undefined &&
            url !== undefined,
        )
        .reduce((map, [, shortcode, url]) => {
          map.set(shortcode, url);
          return map;
        }, new Map()),
    );
  }

  // URLをリンクに変換する関数
  function convertUrlToLink(url: any) {
    return `<a class="anchor" href="${url}" target="_blank">${url}</a>`;
  }

  // 画像URLを画像に変換する関数
  function convertImageUrlToImage(url: any, style: string) {
    return `<img src="${url}" alt="" style="${style}"/>`;
  }

  function getImageSrc(url: string) {
    const ext = url.split('.').pop()?.toLowerCase();
    if (
      ext === 'jpg' ||
      ext === 'jpeg' ||
      ext === 'png' ||
      ext === 'gif' ||
      ext === 'webp'
    ) {
      return url;
    }
    return '';
  }

  // noteを表示用に変換
  let convertedNote: string | undefined = undefined; // 初期値を明示的に設定
  const emojiRegex = /(:[^\s:]+:)/g;

  $: if (note?.content.length != 0) {
    convertedNote = sanitizeHtml(note?.content as string, {
      allowedTags: allowedTags,
      allowedAttributes: allowedAttributes,
    })
      .split(urlRegex)
      .map((part: string) => {
        if (part.match(urlRegex)) {
          if (part.includes('http://') || part.includes('https://')) {
            if (getImageSrc(part)) {
              return convertImageUrlToImage(
                part,
                `display: inline; max-height: 10em;`,
              );
            } else {
              return convertUrlToLink(part);
            }
          }
          return part;
        } else {
          return part;
        }
      })
      .join('');
  }

  // emojisの要素がある場合にshortcodeをURL画像に置換
  $: if (convertedNote?.length != 0 && emojis.size > 0) {
    convertedNote = convertedNote?.replace(emojiRegex, (match) => {
      const shortcode = match.slice(1, -1);
      const imageUrl = emojis.get(shortcode);
      if (imageUrl) {
        return convertImageUrlToImage(
          imageUrl,
          `display: inline;max-height: 1.5em;`,
        );
      }
      return match;
    });
  }

  function handleClick(event: { target: any }) {
    const clickedElement = event.target;
    if (clickedElement.tagName === 'IMG') {
      // 画像がクリックされた場合の処理
      const imageUrl = clickedElement.getAttribute('src');
      const modal = {
        type: 'component' as const,
        image: imageUrl,
        component: modalComponent,
      };
      modalStore.trigger(modal);

      console.log('Image clicked!');
      console.log('Image URL:', imageUrl);
      // ここに独自の処理を追加します
    } else if (clickedElement.tagName === 'A') {
      // リンクがクリックされた場合の処理
      console.log('Link clicked!');
      // ここに独自の処理を追加します
    } else {
      // その他の要素がクリックされた場合の処理
      console.log('Element clicked!');
      // ここに独自の処理を追加します
    }
  }

  function handleClickPubkey() {
    const modal = {
      backdropClasses:
        '!bg-surface-400 dark:!bg-surface-700  !bg-opacity-10 dark:!bg-opacity-10',

      type: 'component' as const,
      //  flyX: x,
      //  flyY: y,
      value: {
        //    position: `x-${clientX} y-${clientY}`,
        hexKey: note?.pubkey,
        pubKey: nip19.npubEncode(note?.pubkey as string),
      },
      component: pubkeyModalComponent,
    };
    modalStore.trigger(modal);
  }
</script>

{#if tag.length > 0}
  <section class="w-full text-left">
    {#if !note}
      <div class="break-all">
        failed to get event ID:{nip19.noteEncode(tag[1])}
      </div>
    {:else if !profile}
      <div class="grid grid-rows-[auto-auto-auto] gap-2 break-all">
        <div>
          <button
            class="underline decoration-1"
            on:click={() => {
              handleClickPubkey();
            }}>{nip19.npubEncode(note?.pubkey)}</button
          >
        </div>
        <div class="break-all whitespace-pre-wrap">{note?.content}</div>
        <div>{new Date(note?.created_at * 1000).toLocaleString()}</div>
      </div>
    {:else}
      <div class="w-full grid grid-cols-[auto_1fr] gap-1">
        <div>
          <Avatar src={content.picture} width="w-12" rounded="rounded-full" />
        </div>
        <div class="grid grid-rows-[auto-auto-auto] gap-2 break-all w-full">
          <div class="w-full grid grid-cols-[auto_1fr_auto] gap-1">
            <div class="font-bold wi truncate ...">{content.display_name}</div>
            <div class="truncate ... wid">
              <button
                class="text-emerald-800 text-sm"
                on:click={() => {
                  handleClickPubkey();
                }}>@<u>{content.name}</u></button
              >
            </div>
            <div class="place-self-end min-w-max">
              {new Date(note?.created_at * 1000).toLocaleString()}
            </div>
          </div>
          <!-- svelte-ignore a11y-click-events-have-key-events -->
          <div class="break-all whitespace-pre-wrap" on:click={handleClick}>
            {@html convertedNote}
          </div>
        </div>
      </div>
    {/if}
  </section>
{/if}

<style>
  .wid {
    min-width: 4em;
  }
</style>
