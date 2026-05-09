// ✅ الحل المبتكر: Enum Dispatch + Generic Wrapper

// أولاً: نعرّف الـ trait للخوارزميات
pub trait ProcessingAlgorithm {
    fn process(&self, data: &[u32]) -> u32;
    fn name(&self) -> &'static str;
}

// ثانياً: الخوارزميات الفعلية
pub struct XorFoldAlgo;
pub struct CrcAlgo;
pub struct Fnv1aAlgo;

impl ProcessingAlgorithm for XorFoldAlgo {
    #[inline(always)]  // نجبر الـ compiler على الـ inlining
    fn process(&self, data: &[u32]) -> u32 {
        data.iter().fold(0u32, |a, &b| a ^ b)
    }
    fn name(&self) -> &'static str { "xor_fold" }
}

impl ProcessingAlgorithm for CrcAlgo {
    #[inline(always)]
    fn process(&self, data: &[u32]) -> u32 {
        // CRC implementation
        data.iter().fold(0xFFFF_FFFFu32, |crc, &b| {
            (crc >> 8) ^ CRC_TABLE[((crc ^ b) & 0xFF) as usize]
        })
    }
    fn name(&self) -> &'static str { "crc32" }
}

// ثالثاً: Enum يجمعهم — هنا السحر
// الـ compiler بيشوف كل الـ variants وقت compile → static dispatch
#[derive(Clone)]
pub enum AlgoKind {
    XorFold(XorFoldAlgo),
    Crc(CrcAlgo),
    Fnv1a(Fnv1aAlgo),
}

impl AlgoKind {
    #[inline(always)]
    pub fn process(&self, data: &[u32]) -> u32 {
        match self {
            // كل match arm بيتحول لـ direct call مش vtable lookup
            AlgoKind::XorFold(a) => a.process(data),
            AlgoKind::Crc(a) => a.process(data),
            AlgoKind::Fnv1a(a) => a.process(data),
        }
    }
}

// رابعاً: الـ Engine اللي بيسمح بالتبديل Runtime
pub struct ProcessingEngine {
    algo: std::sync::Arc<std::sync::RwLock<AlgoKind>>,
}

impl ProcessingEngine {
    pub fn new(algo: AlgoKind) -> Self {
        Self { algo: std::sync::Arc::new(std::sync::RwLock::new(algo)) }
    }

    // Hot path — read lock فقط، مفيش write
    #[inline]
    pub fn run(&self, data: &[u32]) -> u32 {
        // read() رخيص جداً — multiple threads ممكن يقروا في نفس الوقت
        self.algo.read().unwrap().process(data)
    }

    // Cold path — نبدّل الخوارزمية في أي وقت
    pub fn swap_algorithm(&self, new_algo: AlgoKind) {
        *self.algo.write().unwrap() = new_algo;
        // الـ write lock هيستنى لحد ما كل الـ reads خلصت
    }
}

// الاستخدام
let engine = ProcessingEngine::new(AlgoKind::XorFold(XorFoldAlgo));

// شغّل على ملايين records
records.par_iter().map(|r| engine.run(&r.data)).collect::<Vec<_>>();

// في أي وقت تبدّل الخوارزمية — حتى تحت Load
engine.swap_algorithm(AlgoKind::Crc(CrcAlgo));
// ✅ الحل المبتكر: Enum Dispatch + Generic Wrapper

// أولاً: نعرّف الـ trait للخوارزميات
pub trait ProcessingAlgorithm {
    fn process(&self, data: &[u32]) -> u32;
    fn name(&self) -> &'static str;
}

// ثانياً: الخوارزميات الفعلية
pub struct XorFoldAlgo;
pub struct CrcAlgo;
pub struct Fnv1aAlgo;

impl ProcessingAlgorithm for XorFoldAlgo {
    #[inline(always)]  // نجبر الـ compiler على الـ inlining
    fn process(&self, data: &[u32]) -> u32 {
        data.iter().fold(0u32, |a, &b| a ^ b)
    }
    fn name(&self) -> &'static str { "xor_fold" }
}

impl ProcessingAlgorithm for CrcAlgo {
    #[inline(always)]
    fn process(&self, data: &[u32]) -> u32 {
        // CRC implementation
        data.iter().fold(0xFFFF_FFFFu32, |crc, &b| {
            (crc >> 8) ^ CRC_TABLE[((crc ^ b) & 0xFF) as usize]
        })
    }
    fn name(&self) -> &'static str { "crc32" }
}

// ثالثاً: Enum يجمعهم — هنا السحر
// الـ compiler بيشوف كل الـ variants وقت compile → static dispatch
#[derive(Clone)]
pub enum AlgoKind {
    XorFold(XorFoldAlgo),
    Crc(CrcAlgo),
    Fnv1a(Fnv1aAlgo),
}

impl AlgoKind {
    #[inline(always)]
    pub fn process(&self, data: &[u32]) -> u32 {
        match self {
            // كل match arm بيتحول لـ direct call مش vtable lookup
            AlgoKind::XorFold(a) => a.process(data),
            AlgoKind::Crc(a) => a.process(data),
            AlgoKind::Fnv1a(a) => a.process(data),
        }
    }
}

// رابعاً: الـ Engine اللي بيسمح بالتبديل Runtime
pub struct ProcessingEngine {
    algo: std::sync::Arc<std::sync::RwLock<AlgoKind>>,
}

impl ProcessingEngine {
    pub fn new(algo: AlgoKind) -> Self {
        Self { algo: std::sync::Arc::new(std::sync::RwLock::new(algo)) }
    }

    // Hot path — read lock فقط، مفيش write
    #[inline]
    pub fn run(&self, data: &[u32]) -> u32 {
        // read() رخيص جداً — multiple threads ممكن يقروا في نفس الوقت
        self.algo.read().unwrap().process(data)
    }

    // Cold path — نبدّل الخوارزمية في أي وقت
    pub fn swap_algorithm(&self, new_algo: AlgoKind) {
        *self.algo.write().unwrap() = new_algo;
        // الـ write lock هيستنى لحد ما كل الـ reads خلصت
    }
}

// الاستخدام
let engine = ProcessingEngine::new(AlgoKind::XorFold(XorFoldAlgo));

// شغّل على ملايين records
records.par_iter().map(|r| engine.run(&r.data)).collect::<Vec<_>>();

// في أي وقت تبدّل الخوارزمية — حتى تحت Load
engine.swap_algorithm(AlgoKind::Crc(CrcAlgo));
