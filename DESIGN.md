# Rally Adventure: Wild Trails — cetak biru ringkas

## BAGIAN 1: GAME DESIGN DOCUMENT (GDD)

### Premis dan core gameplay
Pemain adalah pereli yang menempuh jalur liar dengan mobil reli tua. Loop inti: **kendalikan mobil → pilih jalur aman/loot → kelola bensin dan kondisi mobil → capai target atau pecahkan rekor → belanjakan koin untuk upgrade**. Kendali bergaya arcade: kemudi punya sedikit inersia, mobil dapat meluncur saat berpindah jalur, dan permukaan lumpur mengurangi traksi. Gas berjalan otomatis agar fokus tetap pada menghindar dan membaca lintasan. Batu memberi kerusakan besar; lumpur merusak mobil dan memangkas kecepatan. Fuel turun terus selama perjalanan, sementara jeriken memberi kesempatan untuk memperpanjang run.

### Tiga mode petualangan
- **Survival Trail:** jarak maksimum sebelum fuel habis atau HP mobil mencapai nol. Rekor jarak menjadi sasaran utama.
- **Lost Compass:** navigasi ke pos pemeriksaan dengan kompas/bearing dan petunjuk di minimap. Jalur yang dipilih menentukan paparan bahaya; mencapai checkpoint menyelesaikan run.
- **Extreme Weather:** badai pasir atau salju membatasi visibilitas, sehingga pemain perlu membaca siluet dan garis jalan lebih awal. Kondisi cuaca dapat berubah sepanjang run.

### Progresi dan ekonomi
Koin yang ditemukan di jalan menjadi mata uang permanen. Upgrade bertingkat (maksimum level 5): **Mesin** meningkatkan kecepatan jelajah; **Ban Lumpur** memperbaiki traksi dan mengurangi selip; **Suspensi** mengurangi kerusakan dari benturan. Harga naik per level untuk menjaga pilihan upgrade tetap bermakna. Jarak terbaik membuka **Gurun (3 km)** lalu **Pegunungan Salju (8 km)**; Hutan adalah area awal. Upgrade sebaiknya memberi keunggulan tetapi tidak menghapus ancaman fuel dan tabrakan.

### Visual dan audio
Gaya **low-poly**, bentuk besar yang terbaca pada ukuran layar kecil, palet berbeda per biome (hijau hutan, oker gurun, biru-putih salju), debu/salju partikel, dan animasi jalan perspektif. UI memakai panel gelap transparan, indikator fuel/HP kontras, ikon sederhana, serta tombol sentuh besar. Audio mesin berlapis yang naik mengikuti kecepatan, musik ritmis perkusi/gitar untuk adrenalin, ambience angin per biome, dan SFX pendek untuk koin, jeriken, dan tabrakan. Sediakan mute dan jangan mengandalkan audio untuk memberi informasi penting.

## BAGIAN 2: ARSITEKTUR KODE (UNITY / GODOT)

Pisahkan simulasi kendaraan, spawning, dan state/UI. `CarController` memiliki Rigidbody, menerima input, mengatur kecepatan longitudinal dan grip lateral. `TrailSpawner` memakai pool prefab dan jarak spawn di depan kendaraan, dengan pemilihan lane serta bobot berdasarkan biome. `GameManager` menjadi sumber state tunggal, menerima event pickup/collision, mengurangi resource, mengukur jarak, dan menghentikan run saat kondisi kalah/menang. Gunakan layer collision untuk memisahkan pickup dari obstacle. Potongan berikut adalah kerangka Unity C#; nama field/UI perlu dihubungkan lewat Inspector.

### Car Controller (drift, akselerasi, rem tangan)
```csharp
using UnityEngine;

public class CarController : MonoBehaviour
{
    public Rigidbody body;
    public float acceleration = 18f, maxSpeed = 24f, turnRate = 70f;
    public float lateralGrip = 5f, handbrakeGrip = 1.4f;
    float steer, throttle;
    bool handbrake;

    void Update() {
        steer = Input.GetAxis("Horizontal");
        throttle = Mathf.Max(0, Input.GetAxis("Vertical"));
        handbrake = Input.GetKey(KeyCode.Space);
    }
    void FixedUpdate() {
        float grip = handbrake ? handbrakeGrip : lateralGrip;
        body.AddForce(transform.forward * throttle * acceleration, ForceMode.Acceleration);
        Vector3 local = transform.InverseTransformDirection(body.velocity);
        local.x = Mathf.Lerp(local.x, 0, grip * Time.fixedDeltaTime); // drift saat grip rendah
        body.velocity = transform.TransformDirection(local);
        float speed01 = Mathf.Clamp01(body.velocity.magnitude / maxSpeed);
        transform.Rotate(0, steer * turnRate * speed01 * Time.fixedDeltaTime, 0);
        body.velocity = Vector3.ClampMagnitude(body.velocity, maxSpeed);
    }
}
```

### Procedural Spawner
```csharp
using UnityEngine;

public class TrailSpawner : MonoBehaviour
{
    public Transform player;
    public GameObject[] obstacles, pickups; // batu/lumpur dan fuel/koin
    public float ahead = 70f, interval = 18f;
    float nextZ;
    void Start() => nextZ = player.position.z + 25f;
    void Update() {
        while (nextZ < player.position.z + ahead) {
            int lane = Random.Range(-1, 2);
            bool pickup = Random.value < .42f;
            var list = pickup ? pickups : obstacles;
            var prefab = list[Random.Range(0, list.Length)];
            Vector3 p = new Vector3(lane * 3f, 0, nextZ);
            Instantiate(prefab, p, Quaternion.identity); // ganti Instantiate dengan object pool
            nextZ += interval * Random.Range(.8f, 1.3f);
        }
    }
}
```

### Game Manager (HUD, HP, fuel, jarak)
```csharp
using UnityEngine;
using UnityEngine.UI;

public class GameManager : MonoBehaviour
{
    public Slider fuelBar, hpBar;
    public Text distanceLabel;
    public GameObject gameOverPanel;
    public Transform car;
    public float fuel = 100, hp = 100;
    Vector3 start;
    void Start() { start = car.position; fuelBar.maxValue = hpBar.maxValue = 100; }
    void Update() {
        fuel = Mathf.Max(0, fuel - Time.deltaTime * .5f);
        float km = Vector3.Distance(start, car.position) / 1000f;
        fuelBar.value = fuel; hpBar.value = hp;
        distanceLabel.text = $"{km:0.00} km";
        if (fuel <= 0 || hp <= 0) gameOverPanel.SetActive(true);
    }
    public void Damage(float amount) => hp = Mathf.Max(0, hp - amount);
    public void Refuel(float amount) => fuel = Mathf.Min(100, fuel + amount);
}
```

## BAGIAN 3: PROTOTIPE KODE SIAP PAKAI

Game playable berada di **[`index.html`](index.html)** sebagai satu file mandiri: HTML5 Canvas dan Vanilla JavaScript, tanpa aset/dependensi eksternal. Buka file tersebut langsung di browser modern. Tersedia tiga pilihan mode, area progresif, upgrade lokal tersimpan, mobil top-down bergaya pseudo-3D, rintangan/loot, fuel dan HP, game over/restart, Web Audio SFX, input keyboard, serta tombol sentuh kiri/kanan responsif untuk ponsel dan tablet. Asumsi prototipe: kontrol gas otomatis dan “minimap” Lost Compass direpresentasikan bearing kompas agar antarmuka tetap ringan.
