# Specification
These are loose requirements. They can be implemented what ever way makes sense.

- [ ] Should be able to create an account
	- [ ] Should be able to login
- [ ] Should be able to create an item
	- [ ] Should be able to add pictures of the item
- [ ] Should be able to create a listing (item with intent to sell)
	- [ ] Listings should be able to transition between statuses
		- [ ] Draft, Active, Inactive, Void, Complete
	- [ ] Should be able to activate a listing at a given time
	- [ ] Should be able to deactivate a listing at a given time
- [ ] Should be able to bid on a listing
	- [ ] Two users bidding on the same listing at the same time should be handled
	- [ ] Two different listings should not influence each other
	- [ ] Bidding is not allowed on incorrect statuses
	- [ ] Users cannot bid on their own listing
	- [ ] Users cannot place bids if they are already the leading bidder
- [ ] Should propagate an event on every Listing state transition
	- [ ] Listing activated
	- [ ] Listing Deactivated
	- [ ] Listing deleted
	- [ ] Listing created
	- [ ] Listing complete (Auction finished)

## Listing state transitions

- Created Listings all start in Draft status
- Draft can transition to Active, Inactive, or Void
- Active can transition to Inactive, Void, or Complete
- Inactive can transition to Active, Void, or Complete
- Void and Complete cannot transition. They are terminal.

#### Status intentions
- Draft (listing has just been created and the owner is still curating it) - the item contained is still editable and no bids can be placed. The item is not viewable to anyone but the owner. The listing can be deleted.
- Active (the listing is live and available to be bid on) - the item contained is not editable. Bids can be placed on the item. The listing cannot be deleted.
- Inactive (the listing is live and NOT available to be bid on. The owner can do basic things to it or the item contained as needed) - the item contained is editable, but cannot be removed/ changed. Bids cannot be placed. The listing cannot be deleted.
- Void (the listing has revoked. No further action can be taken.) - the item contained is not editable. Bids cannot be placed and the item is promised to no one. The item and listing can be deleted.
- Complete (the listing has completed. The item is now promised to the winner of the auction) - The item contained is not editable. Bids cannot be placed and the item is promised to the highest bidder. Item and listing cannot be deleted.

