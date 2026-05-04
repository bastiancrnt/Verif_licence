import asyncio
from playwright.async_api import async_playwright
import pandas as pd

# --------- CONFIG
USERNAME = "0985535"
PASSWORD = "Challenge.2026"

CSV_IN = "FFSU.csv"
CSV_OUT = "FFSU_resultats.csv"

SEP = ","      # mets ";" si ton CSV est séparé par des points-virgules
NUM_LEN = 7    # mets None si tu ne veux pas forcer une longueur (sinon 7 pour les zéros)

# --------- UTILS
def norm_num(x) -> str:
    s = str(x).strip()
    if s.lower() in {"nan", "none", ""}:
        return ""
    if s.endswith(".0"):
        s = s[:-2]
    s = s.strip()
    if not s.isdigit():
        return s
    if NUM_LEN is not None:
        s = s.zfill(NUM_LEN)
    return s

async def clear_and_type(locator, text: str):
    await locator.click()
    await locator.press("Control+A")
    await locator.press("Backspace")
    await locator.type(str(text), delay=15)

async def wait_results_updated(page):
    # On attend qu'au moins le tableau existe (même si 0 lignes)
    await page.wait_for_selector("table", timeout=60000)
    # Petit délai UI (le site remplace souvent le tbody)
    await page.wait_for_timeout(250)

async def run():
    # Lire Numero en string pour conserver les zéros initiaux
    df = pd.read_csv(CSV_IN, sep=SEP, dtype={"Numero": str})
    df["Numero"] = df["Numero"].astype(str).str.replace(r"\.0$", "", regex=True)

    if "Statut" not in df.columns:
        df["Statut"] = "Non vérifié"
    if "NumeroTrouve" not in df.columns:
        df["NumeroTrouve"] = ""

    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=False)
        page = await browser.new_page()

        # --- Login
        await page.goto("https://gestion.mysportu.com/auth/login", wait_until="domcontentloaded")
        await page.fill('input[name="username"]', USERNAME)
        await page.fill('input[name="password"]', PASSWORD)
        await page.get_by_role("button", name="Me connecter").click()

        # --- Navigation
        await page.get_by_role("link", name="licences").wait_for(timeout=60000)
        await page.get_by_role("link", name="licences").click()
        await page.wait_for_selector("a[href*='licences/recherche']", timeout=60000)
        await page.click("a[href*='licences/recherche']")
        await page.wait_for_load_state("networkidle")

        print("Page recherche licences prête")

        # Champ réel (d'après ton debug)
        search = page.locator("#search_licencies")
        await search.wait_for(state="visible", timeout=60000)

        rows = page.locator("table tbody tr")

        # --- Fonction robuste: lit numéro + texte complet de ligne (pas de nth(3))
        async def read_row(tr):
            tds = tr.locator("td")
            m = await tds.count()
            if m == 0:
                return "", ""
            num_cell = (await tds.first.inner_text()).strip()
            row_text = (await tr.inner_text()).strip()
            return norm_num(num_cell), row_text

        for i, r in df.iterrows():
            nom = str(r["Nom"]).strip()
            numero_raw = norm_num(r["Numero"])  # peut être "Aucune license" ou un vrai numéro

            # --- Recherche par nom
            await clear_and_type(search, nom)
            await page.keyboard.press("Enter")
            await wait_results_updated(page)

            n = await rows.count()
            if n == 0:
                df.loc[i, "Statut"] = "Nom introuvable"
                continue

            # --- Cas 1: on a un numéro exploitable => on matche la bonne ligne par numéro
            if numero_raw.isdigit():
                numero = numero_raw
                found = False
                for k in range(n):
                    tr = rows.nth(k)
                    num_cell, row_text = await read_row(tr)
                    if num_cell == numero:
                        df.loc[i, "NumeroTrouve"] = num_cell
                        df.loc[i, "Statut"] = "Active" if "Active" in row_text else "Non active"
                        found = True
                        break

                if not found:
                    df.loc[i, "Statut"] = "Numéro non trouvé (mais nom trouvé)"
                continue

            # --- Cas 2: numéro absent/texte (ex: "Aucune license")
            # => on choisit une ligne "Active" si unique, sinon ambigu/aucune
            active_nums = []
            for k in range(n):
                tr = rows.nth(k)
                num_cell, row_text = await read_row(tr)
                if "Active" in row_text:
                    active_nums.append(num_cell)

            if len(active_nums) == 1:
                df.loc[i, "NumeroTrouve"] = active_nums[0]
                df.loc[i, "Statut"] = "Active"
            elif len(active_nums) > 1:
                df.loc[i, "NumeroTrouve"] = "|".join(active_nums)
                df.loc[i, "Statut"] = "Ambigu (plusieurs Active)"
            else:
                df.loc[i, "Statut"] = "Aucune Active (nom trouvé)"

        df.to_csv(CSV_OUT, sep=SEP, index=False)
        print(f"Résultats écrits dans: {CSV_OUT}")

        await browser.close()

if __name__ == "__main__":
    asyncio.run(run())
