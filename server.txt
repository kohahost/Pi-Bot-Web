import express from 'express';
import dotenv from 'dotenv';
import { Keypair, Server, Networks, TransactionBuilder, Operation, BASE_FEE, Memo } from 'stellar-sdk';
import { mnemonicToSeedSync } from 'bip39';
import crypto from 'crypto';
import axios from 'axios';
import path from 'path';
import { fileURLToPath } from 'url';

dotenv.config();
const app = express();
const port = 3000;
app.use(express.static('public'));
app.use(express.json());

const __dirname = path.dirname(fileURLToPath(import.meta.url));
const server = new Server(process.env.HORIZON_URL);

// 🔐 Generate keypair from mnemonic
function mnemonicToKeypair(mnemonic) {
  const seed = mnemonicToSeedSync(mnemonic);
  const raw = seed.slice(0, 32);
  return Keypair.fromRawEd25519Seed(raw);
}

// 🔔 Send Telegram message
async function sendTelegram(text) {
  try {
    await axios.get(`https://api.telegram.org/bot${process.env.TELEGRAM_BOT_TOKEN}/sendMessage`, {
      params: {
        chat_id: process.env.TELEGRAM_CHAT_ID,
        text
      }
    });
  } catch (e) {
    console.error('Gagal kirim Telegram:', e.message);
  }
}

// 🔁 Main logic: claim + withdraw
async function claimAndWithdraw(mnemonic) {
  const kp = mnemonicToKeypair(mnemonic);
  const sponsor = mnemonicToKeypair(process.env.SPONSOR_MNEMONIC);
  const account = await server.loadAccount(kp.publicKey());
  const sponsorAccount = await server.loadAccount(sponsor.publicKey());

  // Cari claimable balance
  const claimables = await server.claimableBalances()
    .claimant(kp.publicKey())
    .call();

  if (claimables.records.length === 0) {
    return { message: 'Tidak ada saldo yang bisa diklaim.' };
  }

  const claimTx = new TransactionBuilder(sponsorAccount, {
    fee: BASE_FEE,
    networkPassphrase: Networks.PI_MAINNET
  });

  for (const c of claimables.records) {
    claimTx.addOperation(Operation.claimClaimableBalance({ balanceId: c.id, source: kp.publicKey() }));
  }

  claimTx.setTimeout(30);
  const built = claimTx.build();
  built.sign(sponsor);
  built.sign(kp);
  await server.submitTransaction(built);

  await new Promise(r => setTimeout(r, 3000));

  // Kirim semua saldo ke RECEIVER
  const updated = await server.loadAccount(kp.publicKey());
  const balance = updated.balances.find(b => b.asset_type === 'native');
  const amount = parseFloat(balance?.balance || '0') - 0.01;

  if (amount <= 0) {
    return { message: 'Saldo terlalu kecil untuk dikirim.' };
  }

  const tx = new TransactionBuilder(updated, {
    fee: BASE_FEE,
    networkPassphrase: Networks.PI_MAINNET
  })
    .addOperation(Operation.payment({
      destination: process.env.RECEIVER_ADDRESS,
      asset: {
        getCode: () => 'XLM',
        getIssuer: () => '',
        isNative: () => true
      },
      amount: amount.toFixed(7)
    }))
    .addMemo(Memo.text("AutoWithdraw"))
    .setTimeout(30)
    .build();

  tx.sign(kp);
  await server.submitTransaction(tx);

  await sendTelegram(`✅ Klaim & kirim Pi berhasil: ${amount} XLM dari ${kp.publicKey().slice(0, 6)}...`);

  return { message: `Berhasil klaim dan kirim ${amount} Pi.` };
}

app.post('/claim', async (req, res) => {
  const { mnemonic } = req.body;
  try {
    const result = await claimAndWithdraw(mnemonic);
    res.json(result);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.listen(port, () => {
  console.log(`✅ Server jalan di http://localhost:${port}`);
});
