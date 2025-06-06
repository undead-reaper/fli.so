<script lang="ts">
  import { fly, fade } from "svelte/transition";
  import { env } from "$env/dynamic/public";
  import {
    Pencil,
    Trash,
    ExternalLink,
    MousePointerClick,
    Earth,
    Lock,
    Copy,
    QrCode,
    Clock,
  } from "lucide-svelte";
  import type { UrlsResponseWithTags } from "$lib/types";
  import { Button } from "$lib/components/ui/button";
  import { initKeyboardShortcuts } from "$lib/keyboard";
  import { cn } from "$lib/utils";
  import { getTagStyles } from "$lib/utils";
  import { toast } from "svelte-sonner";
  import { QRCodeDialog } from "$lib/components/ui/core/misc";
  import type { TailwindColor } from "$lib/types/colors";

  interface Props {
    url: UrlsResponseWithTags;
    onEdit: (url: UrlsResponseWithTags) => void;
    onDelete: (id: string) => void;
  }

  let { url, onEdit, onDelete }: Props = $props();
  let hoveredUrl = $state<string | null>(null);
  let showDeleteConfirm = $state(false);
  let showQRDialog = $state(false);
  let isExpired = $derived(
    url.expiration && new Date(url.expiration) < new Date(),
  );

  // Handle keyboard shortcuts when card is hovered
  $effect(() => {
    if (hoveredUrl === url.id) {
      return initKeyboardShortcuts([
        { key: "e", handler: () => !showDeleteConfirm && onEdit(url) },
        {
          key: "d",
          handler: () => !showDeleteConfirm && (showDeleteConfirm = true),
        },
        { key: "Escape", handler: () => (showDeleteConfirm = false) },
        {
          key: "q",
          handler: () => (showQRDialog = true),
        },
        {
          key: "c",
          handler: () =>
            navigator.clipboard
              .writeText(`${env.PUBLIC_APPLICATION_URL}/${url.slug}`)
              .then(() => toast.success("Copied to clipboard")),
        },
      ]);
    }
  });

  function handleDelete() {
    showDeleteConfirm = false;
    onDelete(url.id);
  }
</script>

<div
  class={cn(
    "rounded-4xl group relative overflow-hidden bg-white/80 p-4 shadow-mild backdrop-blur-sm transition-all duration-200 hover:translate-y-[-2px] hover:cursor-pointer hover:bg-white hover:shadow-subtle dark:bg-slate-800/80 dark:hover:bg-slate-800 dark:hover:shadow-slate-900/50",
    isExpired && "grayscale-[0.5] hover:grayscale-0",
  )}
  in:fly|local={{ y: 10, duration: 200, delay: 50 }}
  out:fade|local={{ duration: 150 }}
  onmouseenter={() => (hoveredUrl = url.id)}
  onmouseleave={() => {
    hoveredUrl = null;
    showDeleteConfirm = false;
  }}
  aria-label={`${url.url} (${url.clicks} clicks)`}
  role="article"
>
  <div class="space-y-3">
    <!-- URL Info -->
    <div class="flex items-center justify-between gap-2">
      <div class="relative">
        <div
          class={cn(
            "group flex size-10 items-center justify-center rounded-full bg-gray-100 text-gray-500 transition-colors duration-200 group-hover:bg-amber-50 group-hover:text-amber-950",
            isExpired &&
              "bg-gray-200 group-hover:bg-red-50 group-hover:text-red-950",
          )}
        >
          <Earth class="size-5" />
        </div>
        {#if url.password_hash}
          <div
            class="absolute -bottom-1 -right-1 rounded-full bg-white p-0.5 shadow-sm dark:bg-slate-800"
          >
            <Lock class="size-3.5 text-gray-500 group-hover:text-amber-950" />
          </div>
        {/if}
        {#if url.expiration}
          <div
            class="absolute -right-1 -top-1 rounded-full bg-white p-1 shadow-sm dark:bg-slate-800"
          >
            <Clock
              class="size-3.5 text-gray-500 group-hover:text-amber-950 dark:group-hover:text-amber-400"
            />
          </div>
        {/if}
      </div>
      <span
        class="inline-flex items-center gap-1 rounded-full bg-input/20 px-2 py-0.5 text-xs font-medium text-slate-600 dark:bg-slate-700 dark:text-slate-300"
      >
        <MousePointerClick class="size-3" />
        {url.clicks} clicks
      </span>
    </div>

    <!-- Short URL -->
    <div class="group/link flex items-start justify-between gap-4">
      <a
        href={`${env.PUBLIC_APPLICATION_URL}/${url.slug}`}
        target="_blank"
        class={cn(
          "group/link flex items-center gap-2 font-medium text-slate-900 group-hover/link:text-amber-600 dark:text-slate-100 dark:group-hover/link:text-gray-900",
          isExpired &&
            "text-slate-500 group-hover/link:text-red-600 dark:text-slate-400 dark:group-hover/link:text-red-400",
          "transition-colors duration-150 ease-out",
        )}
      >
        <div class="line-clamp-1">
          {env.PUBLIC_APPLICATION_NAME}/{url.slug}
          {#if isExpired}
            <span class="text-xs text-red-500 dark:text-red-400">(Expired)</span
            >
          {/if}
        </div>

        <ExternalLink
          class="h-4 w-4 opacity-0 transition-opacity group-hover/link:opacity-100"
        />
      </a>
    </div>

    <!-- Original URL -->
    <p
      class="truncate text-sm text-slate-500 dark:text-slate-400"
      title={url.url}
    >
      {url.url}
    </p>

    <!-- Tags and Actions -->
    <div class="relative flex items-center justify-end gap-2">
      <!-- Tags -->
      {#if url.expand?.tags_id}
        <div
          class="group/tags relative min-w-0 flex-1 transition-all duration-300 ease-out"
          class:translate-x-[-150%]={showDeleteConfirm}
          class:opacity-0={showDeleteConfirm}
        >
          <!-- When not hovered, show all tags -->
          <div
            class="flex items-center gap-x-1 transition-opacity duration-200 group-hover:opacity-0"
          >
            {#each url.expand.tags_id as tag}
              {@const styles = getTagStyles(tag.color as TailwindColor)}
              <span
                class={cn(
                  "inline-flex max-w-[150px] items-center gap-1 rounded-full border-[0.5px] border-gray-200  px-2 py-0.5 text-xs first:ml-0 hover:z-10",
                  styles.text,
                )}
              >
                <span class={cn("h-2 w-2 shrink-0 rounded-full", styles.dot)}
                ></span>
                <span class="truncate">{tag.name}</span>
              </span>
            {/each}
          </div>

          <!-- When hovered, show only first tag + count -->
          <div
            class="absolute inset-0 flex items-center opacity-0 transition-opacity duration-200 group-hover:opacity-100"
          >
            {#if url.expand.tags_id.length > 0}
              {@const styles = getTagStyles(
                url.expand.tags_id[0].color as TailwindColor,
              )}
              <span
                class={cn(
                  "inline-flex max-w-[150px] items-center gap-1 rounded-full border-[0.5px] border-gray-200  px-2 py-0.5 text-xs",
                  styles.text,
                )}
              >
                <span class={cn("h-2 w-2 shrink-0 rounded-full", styles.dot)}
                ></span>
                <span class="truncate">{url.expand.tags_id[0].name}</span>
                {#if url.expand.tags_id.length > 1}
                  <span class="ml-1 shrink-0 text-slate-500"
                    >+{url.expand.tags_id.length - 1}</span
                  >
                {/if}
              </span>
            {/if}
          </div>
        </div>
      {/if}

      <!-- Action Buttons -->
      <div
        class="ml-auto flex items-center gap-2 transition-all duration-300 ease-out"
        class:translate-x-[-150%]={showDeleteConfirm}
        class:opacity-0={showDeleteConfirm}
        class:invisible={showDeleteConfirm}
      >
        <Button
          id={`copy-${url.id}`}
          on:click={() =>
            navigator.clipboard
              .writeText(`${env.PUBLIC_APPLICATION_URL}/${url.slug}`)
              .then(() => toast.success("Copied to clipboard"))}
          class="group/btn relative rounded-full p-2 text-gray-600 opacity-0 transition-opacity duration-200 hover:bg-gray-100 group-hover:opacity-100 dark:text-gray-400 dark:hover:bg-gray-800"
          variant="ghost"
          size="icon"
          title="Copy URL (press 'c')"
        >
          <Copy class="h-4 w-4" />
        </Button>

        <Button
          id={`qr-${url.id}`}
          on:click={() => (showQRDialog = true)}
          class="group/btn relative rounded-full p-2 text-gray-600 opacity-0 transition-opacity duration-200 hover:bg-gray-100 group-hover:opacity-100 dark:text-gray-400 dark:hover:bg-gray-800"
          variant="ghost"
          size="icon"
          title="Show QR Code (press 'q')"
        >
          <QrCode class="h-4 w-4" />
        </Button>

        <Button
          id={`edit-${url.id}`}
          on:click={() => onEdit(url)}
          class="group/btn relative rounded-full p-2 text-gray-600 opacity-0 transition-opacity duration-200 hover:bg-gray-100 group-hover:opacity-100 dark:text-gray-400 dark:hover:bg-gray-800"
          variant="ghost"
          size="icon"
          title="Edit URL (press 'e')"
        >
          <Pencil class="h-4 w-4" />
        </Button>

        <Button
          id={`delete-${url.id}`}
          onclick={() => (showDeleteConfirm = true)}
          class="group/btn relative rounded-full p-2 text-red-500 opacity-0 transition-opacity duration-200 hover:bg-red-50 hover:text-red-700 group-hover:opacity-100 dark:text-red-400 dark:hover:bg-red-950/50 dark:hover:text-red-300"
          title="Delete URL (press 'd')"
          size="icon"
          variant="ghost"
        >
          <Trash class="h-4 w-4" />
        </Button>
      </div>

      <!-- Delete Confirmation -->
      <div
        class="absolute right-0 flex transform items-center gap-2 transition-all duration-300 ease-out"
        class:translate-x-0={showDeleteConfirm}
        class:translate-x-[150%]={!showDeleteConfirm}
        class:opacity-100={showDeleteConfirm}
        class:opacity-0={!showDeleteConfirm}
        class:pointer-events-none={!showDeleteConfirm}
      >
        <Button
          onclick={() => (showDeleteConfirm = false)}
          variant="ghost"
          size="sm"
          class="text-slate-600 hover:bg-slate-100 dark:hover:bg-slate-700"
        >
          Cancel (Esc)
        </Button>
        <Button
          onclick={handleDelete}
          variant="destructive"
          size="sm"
          class="bg-red-500 hover:bg-red-600 dark:bg-red-600 dark:hover:bg-red-700"
        >
          Delete
        </Button>
      </div>
    </div>
  </div>
</div>

<QRCodeDialog
  url={`${env.PUBLIC_APPLICATION_URL}/${url.slug}`}
  open={showQRDialog}
  onOpenChange={(open) => (showQRDialog = open)}
/>
