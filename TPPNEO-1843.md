Update Expiration is allowed via API/Bulk but blocked via UI when Prepaid fixed contract is assigned


fix contract: không cho update expiration
var contract : cho update nhưng không được là quá khứ (sai)
var contract : cho update nhưng không được là thời điểm trước expiration hiện tại, ví dụ: expiration hiện tại của device là ngày 03/02/2026 11:15:00 thì không được update về 02/02/2026 11:15:00 trở về trước, ngày hiện tại là 30/01/2026 11:19:00 (đúng)