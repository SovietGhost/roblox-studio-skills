---
name: roblox-monetization
description: Implementing Roblox monetization correctly — developer products with idempotent ProcessReceipt, game passes, subscriptions, Premium benefits, private servers, price-checking and purchase prompts, and ethical/compliance guardrails. Use when adding shops, purchases, passes, or any Robux flow.
---

# Monetization

## Developer products — ProcessReceipt (critical)

Rules:
- Exactly **one** `MarketplaceService.ProcessReceipt` callback in the whole game (assigning it twice overwrites).
- Return `Enum.ProductPurchaseDecision.PurchaseGranted` **only after** the grant is durably saved. Otherwise return `NotProcessedYet` — Roblox retries later (including on next join).
- Idempotency: track `receiptInfo.PurchaseId`; if already processed, return `PurchaseGranted` without granting again.
- Player may have left: if `Players:GetPlayerByUserId(receiptInfo.PlayerId)` is nil, return `NotProcessedYet`.

```lua
--!strict
local handlers: { [number]: (Player, any) -> () } = {
	[123456] = function(player, profileData) profileData.coins += 1000 end,
}

MarketplaceService.ProcessReceipt = function(receipt)
	local player = Players:GetPlayerByUserId(receipt.PlayerId)
	if not player then return Enum.ProductPurchaseDecision.NotProcessedYet end
	local data = PlayerDataService.GetLoaded(player) -- must be session-locked profile data
	if not data then return Enum.ProductPurchaseDecision.NotProcessedYet end

	if data.purchases[receipt.PurchaseId] then
		return Enum.ProductPurchaseDecision.PurchaseGranted
	end
	local handler = handlers[receipt.ProductId]
	if not handler then
		warn(`Unknown product {receipt.ProductId}`)
		return Enum.ProductPurchaseDecision.NotProcessedYet
	end
	local ok, err = pcall(handler, player, data)
	if not ok then warn(err); return Enum.ProductPurchaseDecision.NotProcessedYet end

	data.purchases[receipt.PurchaseId] = true
	-- Persist now (force save / profile save) before granting the decision.
	if not PlayerDataService.SaveNow(player) then
		return Enum.ProductPurchaseDecision.NotProcessedYet
	end
	return Enum.ProductPurchaseDecision.PurchaseGranted
end
```

Cap the `purchases` history (keep last ~50–100 ids) so player data doesn't grow unbounded. With ProfileStore,
follow its documented receipt pattern (it handles the save-before-grant via its own mechanism).

## Game passes

- Check ownership on join: `pcall(MarketplaceService.UserOwnsGamePassAsync, MarketplaceService, player.UserId, passId)`; cache result per session.
- Listen to `MarketplaceService.PromptGamePassPurchaseFinished` (server) to grant immediately after in-game purchase (`wasPurchased == true`) — then re-verify ownership.
- Don't use DataStore as the source of truth for pass ownership — Roblox is. Re-check `UserOwnsGamePassAsync` each session.

## Prompts

- Client or server can call `MarketplaceService:PromptProductPurchase(player, id)` / `PromptGamePassPurchase`. Server-side prompts are safer for gated flows.
- Display prices from `MarketplaceService:GetProductInfoAsync(id, Enum.InfoType.Product)` (pcall, cache) — don't hardcode Robux prices in UI.
- Disable the buy button while a prompt is open.

## Subscriptions & Premium

- Subscriptions: `MarketplaceService:GetUserSubscriptionStatusAsync(player, subscriptionId)`, `UserSubscriptionStatusChanged` event. Grant benefits while active.
- Premium: `player.MembershipType == Enum.MembershipType.Premium`; `Players.PlayerMembershipChanged`. Premium payouts reward engagement; Premium perks are optional.

## Testing

Studio purchases are simulated (no Robux charged) — test grant, duplicate receipt (call handler twice with same PurchaseId), player-left, and save-failure paths.

## Policy / ethics guardrails

- Respect `PolicyService:GetPolicyInfoForPlayerAsync(player)` — e.g. `ArePaidRandomItemsRestricted` (loot boxes), `IsPaidItemTradingAllowed`. Hide restricted features for those players.
- Paid random items must disclose odds.
- No misleading "limited time" pressure or dark patterns targeting minors; follow Roblox Community Standards and advertising rules.
