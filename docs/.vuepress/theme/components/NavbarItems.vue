<script setup lang="ts">
import AutoLink from "@vuepress/theme-default/lib/client/components/AutoLink.vue";
import NavbarDropdown from "@vuepress/theme-default/lib/client/components/NavbarDropdown.vue";
import { useRouteLocale, useSiteLocaleData } from "@vuepress/client";
import { isLinkHttp, isString } from "@vuepress/shared";
import { computed } from "vue";
import type { ComputedRef } from "vue";
import { useRouter } from "vue-router";
import type {
  NavbarGroup,
  NavbarItem,
  ResolvedNavbarItem,
} from "@vuepress/theme-default/lib/shared";
import {
  useNavLink,
  useThemeLocaleData,
} from "@vuepress/theme-default/lib/client";
import { resolveRepoType } from "@vuepress/theme-default/lib/client/utils";

/**
 * Get navbar config of select language dropdown
 */
const useNavbarSelectLanguage = (): ComputedRef<ResolvedNavbarItem[]> => {
  const router = useRouter();
  const routeLocale = useRouteLocale();
  const siteLocale = useSiteLocaleData();
  const themeLocale = useThemeLocaleData();

  return computed<ResolvedNavbarItem[]>(() => {
    const localePaths = Object.keys(siteLocale.value.locales);
    // do not display language selection dropdown if there is only one language
    if (localePaths.length < 2) {
      return [];
    }
    const currentPath = router.currentRoute.value.path;
    const currentFullPath = router.currentRoute.value.fullPath;
    const currentHash = router.currentRoute.value.hash;

    const languageDropdown: ResolvedNavbarItem = {
      text: themeLocale.value.selectLanguageText ?? "unknown language",
      ariaLabel:
        themeLocale.value.selectLanguageAriaLabel ??
        themeLocale.value.selectLanguageText ??
        "unknown language",
      children: localePaths.map((targetLocalePath) => {
        // target locale config of this language link
        const targetSiteLocale =
          siteLocale.value.locales?.[targetLocalePath] ?? {};
        const targetThemeLocale =
          themeLocale.value.locales?.[targetLocalePath] ?? {};
        const targetLang = `${targetSiteLocale.lang}`;

        const text = targetThemeLocale.selectLanguageName ?? targetLang;
        let link;

        if (targetLang === siteLocale.value.lang) {
          // if the target language is current language
          // stay at current link
          link = currentFullPath;
        } else {
          // if the target language is not current language
          // try to link to the corresponding page of current page
          // or fallback to homepage
          const targetLocalePage = currentPath.replace(
            routeLocale.value,
            targetLocalePath
          );
          if (
            router.getRoutes().some((item) => item.path === targetLocalePage)
          ) {
            // try to keep current hash across languages
            link = `${targetLocalePage}${currentHash}`;
          } else {
            link = targetThemeLocale.home ?? targetLocalePath;
          }
        }

        return {
          text,
          link,
        };
      }),
    };

    return [languageDropdown];
  });
};

/**
 * Get navbar config of repository link
 */
const useNavbarRepo = (): ComputedRef<ResolvedNavbarItem[]> => {
  const themeLocale = useThemeLocaleData();

  const repo = computed(() => themeLocale.value.repo);
  const repoType = computed(() =>
    repo.value ? resolveRepoType(repo.value) : null
  );

  const repoLink = computed(() => {
    if (repo.value && !isLinkHttp(repo.value)) {
      return `https://github.com/${repo.value}`;
    }

    return repo.value;
  });

  const repoLabel = computed(() => {
    if (!repoLink.value) return null;
    if (themeLocale.value.repoLabel) return themeLocale.value.repoLabel;
    if (repoType.value === null) return "Source";
    return repoType.value;
  });

  return computed(() => {
    if (!repoLink.value || !repoLabel.value) {
      return [];
    }

    return [
      {
        text: repoLabel.value,
        link: repoLink.value,
      },
    ];
  });
};

const resolveNavbarItem = (
  item: NavbarItem | NavbarGroup | string
): ResolvedNavbarItem => {
  if (isString(item)) {
    return useNavLink(item);
  }
  if ((item as NavbarGroup).children) {
    return {
      ...item,
      children: (item as NavbarGroup).children.map(resolveNavbarItem),
    };
  }
  return item as ResolvedNavbarItem;
};

const useNavbarConfig = (): ComputedRef<ResolvedNavbarItem[]> => {
  const themeLocale = useThemeLocaleData();
  return computed(() =>
    (themeLocale.value.navbar || []).map(resolveNavbarItem)
  );
};

const navbarConfig = useNavbarConfig();
const navbarSelectLanguage = useNavbarSelectLanguage();
const navbarRepo = useNavbarRepo();
const navbarLinks = computed(() => [
  ...navbarConfig.value,
  ...navbarSelectLanguage.value,
  ...navbarRepo.value,
]);
</script>

<template>
  <nav v-if="navbarLinks.length" class="navbar-items">
    <div v-for="item in navbarLinks" :key="item.text" class="navbar-item">
      <NavbarDropdown v-if="item.children" :item="item" />
      <AutoLink v-else :item="item" />
    </div>
    <div class="navbar-item nav-button">
      <a href="https://app.umee.cc/" target="_blank"
        ><Button>Launch App</Button></a
      >
    </div>
  </nav>
</template>
